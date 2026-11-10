# 核心底层逻辑伪代码 (公开示例版)

> **免责声明**：以下为示例实现，用于展示逻辑闭环与状态机交互，非最终数值；内部版使用不同步长与保底阈值，参数以内部平衡测试为准。
>
> **示例代码声明**：本示例省略护甲、减伤、无敌帧、死亡结算、法器独立硬直等细节，非生产可用。

## 0. 通用约定

*   **护盾/血量**：示例使用整数定点单位，避免浮点等值判断失效；概率使用浮点。
*   **PRD 粒度**：一次“伤害段”只 Roll 一次 PRD，缓存暴击结果后对命中目标应用。AOE 同一伤害段共享一次 Roll；多段技能每段独立 Roll。
*   **多段伤害**：每段独立调用 `ApplyDamage`；破盾保护只作用于“击碎护盾的那一段”。
*   **并发**：`player.prdState` 的读改写由 per-player 串行队列/锁保护（内部版实现，公开版不展开）。
*   **判定优先级**：`保底 > 心流 > 常规`。
*   **随机源**：`ServerRandom()` 返回 `[0, 1)`；内部版使用可复现种子，公开版不展开。
*   **PVP/PVE**：参数通过 `GetPRDParams(context.mode)` 从配置表读取；跨模式切换时调用 `RecomputePRDChance` 按新参数重算，保留 `misses`。

## 1. 体术暴击（PRD 伪随机保底）算法

**工程说明**：
*   **Server-side only**：暴击判定必须在服务端执行，客户端仅做表现反馈。
*   **Per-player state**：`misses` 和 `currentChance` 绑定到每个玩家实例（`player.prdState`），不能全局共享。
*   **初始化**：玩家创建时调用 `InitPRDState(params)`，使用当前模式参数。
*   **事件级 Roll**：`ResolveMeleeAttackEvent` 对一次伤害段只调用一次 `RollMeleeCrit`，再对多目标应用结果。
*   **Random Source**：使用 `ServerRandom()`，严禁客户端 `Math.random()`。

```text
// 公开示例参数，非最终内部数值
const BASE_CHANCE = 0.05;      // 基础概率 5%
const STEP = 0.02;             // 每次未暴击递增 2%
const PITY_THRESHOLD = 20;     // 连续20次未暴击后，第21拳必出

struct PRDState { int misses; float currentChance; }

// 玩家创建时初始化（参数来自当前模式）
function InitPRDState(params):
    return { misses: 0, currentChance: params.base }

// 示意：PVP 锁定硬核参数；PVE 按难度返回参数
function GetPRDParams(mode):
    return { base: BASE_CHANCE, step: STEP, pity: PITY_THRESHOLD }

// 热更新/切换模式时重算概率，保留保底进度
function RecomputePRDChance(state, params):
    state.currentChance = min(1.0, params.base + state.misses * params.step)

// 事件级入口：一次伤害段只 Roll 一次，再对多目标应用
function ResolveMeleeAttackEvent(player, targets, context):
    if (!context.isMeleeHit) return

    isCrit = RollMeleeCrit(player, context)
    for target in targets:
        if (isCrit):
            TriggerMeleeCrit(player, target, context) // 同人表现：黑闪

// 只更新一次 PRD 状态，返回本次是否暴击
function RollMeleeCrit(player, context):
    state = player.prdState
    params = GetPRDParams(context.mode)

    // 1. 保底优先级最高
    if (state.misses >= params.pity):
        ResetPRD(state, params)
        return true

    // 2. 心流优先级次之：暂停 PRD 累加，由心流模块独立接管
    if (IsInZone(player)):
        if (ServerRandom() < GetZoneChance(player)):
            ResetPRD(state, params) // 心流暴击同样清空保底计数
            return true
        return false

    // 3. 常规状态修正
    effectiveChance = Clamp(
        state.currentChance + GetStateModifier(player, context),
        0.0, 1.0
    )

    // 4. 判定
    if (ServerRandom() < effectiveChance):
        ResetPRD(state, params)
        return true
    else:
        state.misses++
        state.currentChance = min(1.0, params.base + state.misses * params.step)
        return false

// 仅在暴击/保底时重置保底进度
function ResetPRD(state, params):
    state.misses = 0
    state.currentChance = params.base
```

> 注：`currentChance` 可由 `misses` 重算；配置热更新/切换模式时调用 `RecomputePRDChance`，**不要**调用 `ResetPRD`，否则会清空保底进度。

## 2 & 3. 伤害结算统一管线 (Damage Pipeline & Shield Protection)

**工程说明**：
*   **结算优先级**：`真实伤害穿透 -> 能量护盾 -> 特殊法器护盾 -> 本体血量`。
*   **防溢出**：`min(当前护盾, 剩余伤害)` 计算吸收量，防止护盾负数。
*   **破盾减伤顺序**：能量护盾碎裂后，**先对溢出伤害减伤，再进入法器护盾**。法器护盾吸收的是减伤后的溢出伤害。
*   **取整策略**：整数除法使用 64 位安全范围，**向下取整**。
*   **多段伤害**：破盾保护只对击碎护盾那一段的溢出生效；CD 在能量盾碎裂且有溢出时即消耗，不论法器是否吸收。
*   **法器独立**：法器破盾默认不触发能量盾保护 CD，独立结算；`BreakArtifactShield` 内部置 `ArtifactShieldActive = false`。法器独立硬直暂省略。
*   **省略项**：护甲、减伤、无敌帧、死亡结算等细节省略。

```text
// 整数定点：溢出伤害 * 30 / 100（衰减至 30%，向下取整）
const SHIELD_PROTECTION_MULT_NUM = 30;
const SHIELD_PROTECTION_MULT_DEN = 100;

// 按单段伤害调用；多段攻击每段独立调用
function ApplyDamage(source, target, rawDamage, damageType):
    if (rawDamage <= 0) return

    // 1. 真实伤害：最高穿透优先级
    if (damageType == TRUE_DAMAGE):
        safeDamage = min(rawDamage, target.Health)
        ApplyToHealth(target, safeDamage)
        return

    remaining = rawDamage

    // 2. 能量护盾 - 优先级 1
    if (target.EnergyShield > 0):
        absorbed = min(target.EnergyShield, remaining)
        target.EnergyShield -= absorbed
        remaining -= absorbed

        if (target.EnergyShield <= 0):
            target.EnergyShield = 0
            TriggerStun(target, 3s)
            BlockEnergyRegen(target, 3s)

            // 破盾减伤发生在法器护盾之前
            if (remaining > 0 && IsShieldProtectionReady(target)):
                remaining = (remaining * SHIELD_PROTECTION_MULT_NUM) / SHIELD_PROTECTION_MULT_DEN
                StartShieldProtectionCooldown(target)

    // 3. 特殊法器护盾 - 优先级 2（吸收减伤后的溢出）
    if (remaining > 0 && target.ArtifactShieldActive && target.ArtifactShield > 0):
        absorbed = min(target.ArtifactShield, remaining)
        target.ArtifactShield -= absorbed
        remaining -= absorbed

        if (target.ArtifactShield <= 0):
            target.ArtifactShield = 0
            BreakArtifactShield(target)

    // 4. 本体血量
    if (remaining > 0):
        safeDamage = min(remaining, target.Health)
        ApplyToHealth(target, safeDamage)
```

## 4. 边界用例（公开自测表）

| 用例 | 输入 | 预期 |
|---|---|---|
| 保底前一刻 | `misses=19`，未暴击 | 本次正常判定；未暴击后 `misses=20`，下一次必暴击 |
| 保底触发 | `misses=20` | 本次必暴击，`misses` 重置为 0 |
| 刚好打空护盾 | 伤害 = 剩余护盾 | 触发硬直与恢复阻断，但**不消耗**破盾保护 CD |
| 护盾溢出 | 伤害 > 护盾 | 触发硬直，溢出部分 ×30%，消耗保护 CD |
| 溢出被法器全吃 | 能量盾破，减伤后伤害 ≤ 法器护盾 | 保护 CD 已消耗，本体不掉血 |
| 真实伤害 | `TRUE_DAMAGE` | 不碰护盾、不触发硬直、不消耗保护 CD |
| 多段攻击 | 第 1 段破盾，第 2 段 | 仅第 1 段溢出生效减伤；第 2 段在 CD 内吃满 |
```