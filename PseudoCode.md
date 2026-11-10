# 核心底层逻辑伪代码 (公开示例版)

> **免责声明**：以下为示例实现，用于展示逻辑闭环与状态机交互，非最终数值；内部版使用不同步长与保底阈值，参数以内部平衡测试为准。

## 1. 体术暴击（PRD 伪随机保底）算法

**工程说明**：
*   **Server-side only**：暴击判定必须在服务端执行（防止客户端篡改），客户端仅做表现反馈。
*   **Per-player state**：`misses` 和 `chance` 必须作为独立状态绑定到每个玩家实例（如 `player.prdState`），不能作为全局变量。
*   **Random Source**：使用服务端专属随机源 `ServerRandom()`，严禁使用客户端的 `Math.random()`。
*   **PVP/PVE 分表**：PVP 使用独立 PRD 参数表，服务器锁定硬核难度；PVE 受动态难度修正影响。

```text
// 常量定义（此处为公开示例参数，非最终内部数值）
const BASE_CHANCE = 0.05;      // 基础概率 5%
const STEP = 0.02;             // 每次未暴击递增 2%
const PITY_THRESHOLD = 20;     // 连续20次未暴击后，第21拳必出

// 玩家独立状态结构
struct PRDState { int misses; float currentChance; }

OnMeleeAttack(player, target, context):
    // 0. 过滤条件：非体术命中（如远程、空挥）不计入PRD累加与重置
    if (!context.isMeleeHit) return

    state = player.prdState

    // 1. 优先检查保底（如果连续20次未暴击，下一次必定暴击）
    if (state.misses >= PITY_THRESHOLD):
        TriggerBlackFlash(player, target, context)
        ResetPRD(state)
        return

    // 2. 心流状态接管
    if (IsInZone(player)):
        // 心流期间，PRD暂停累加，由心流模块独立接管暴击率
        if (ServerRandom() < GetZoneChance(player)):
            TriggerBlackFlash(player, target, context)
            ResetPRD(state) // 心流期间暴击同样清空保底计数，防止心流结束后白嫖保底
        return

    // 3. 常规状态修正接口（过载/绝境/难度加成在此叠加）
    effectiveChance = Clamp(
        state.currentChance + GetStateModifier(player, context),
        0.0, 1.0
    )

    // 4. 判定触发
    if (ServerRandom() < effectiveChance):
        TriggerBlackFlash(player, target, context)
        ResetPRD(state)
    else:
        // 5. 未暴击，累加概率
        state.misses++
        state.currentChance = Min(1.0, BASE_CHANCE + state.misses * STEP)

// 状态重置函数
function ResetPRD(state):
    state.misses = 0
    state.currentChance = BASE_CHANCE
```

## 2 & 3. 伤害结算统一管线 (Damage Pipeline & Shield Protection)

**工程说明**：
*   **结算优先级**：严格遵循白皮书 3.2 节，即 `真实伤害穿透 -> 能量护盾 -> 特殊法器护盾 -> 本体血量`。
*   **防溢出机制**：使用 `min(当前护盾, 剩余伤害)` 计算吸收量，防止护盾出现负数导致破盾判定失效。
*   **多段伤害判定**：破盾保护（减伤）仅对“击碎护盾的那一段攻击的溢出伤害”生效。若能量护盾碎裂且存在溢出，即使溢出伤害被后续的法器护盾完全吸收，该次破盾保护的 CD 依然会被消耗。
*   **法器独立**：法器护盾破盾默认不触发能量护盾的破盾保护 CD，独立结算。

```text
const SHIELD_PROTECTION_MULT = 0.30; // 破盾溢出伤害 ×0.30（即衰减至 30%）

function ApplyDamage(source, target, rawDamage, damageType):
    if (rawDamage <= 0) return

    // 1. 真实伤害：最高穿透优先级
    // 直接扣血，不碰任何护盾，不触发破盾硬直与保护CD
    if (damageType == TRUE_DAMAGE):
        // 安全防护：防止因计算误差导致直接秒杀或血量溢出
        safeDamage = min(rawDamage, target.Health)
        ApplyToHealth(target, safeDamage)
        return

    remaining = rawDamage

    // 2. 能量护盾 (Energy Shield) - 优先级 1
    if (target.EnergyShield > 0):
        absorbed = min(target.EnergyShield, remaining)
        target.EnergyShield -= absorbed
        remaining -= absorbed

        if (target.EnergyShield == 0):
            TriggerStun(target, 3s) // 触发破盾硬直
            BlockEnergyRegen(target, 3s) // 与硬直同窗口，阻断恢复

            // 核心逻辑：只要有溢出伤害，立刻判定减伤并消耗CD（哪怕后面被法器吸收）
            if (remaining > 0 && IsShieldProtectionReady(target)):
                remaining = remaining * SHIELD_PROTECTION_MULT
                StartShieldProtectionCooldown(target)
            // 如果刚好打空（remaining=0），触发硬直但不消耗保护CD

    // 3. 特殊法器护盾 (Artifact Shield) - 优先级 2
    if (remaining > 0 && target.ArtifactShieldActive && target.ArtifactShield > 0):
        absorbed = min(target.ArtifactShield, remaining)
        target.ArtifactShield -= absorbed
        remaining -= absorbed

        if (target.ArtifactShield == 0):
            BreakArtifactShield(target) // 法器破盾特效/状态失效（独立结算，不影响能量盾CD）

    // 4. 本体血量 (Health)
    if (remaining > 0):
        // 安全防护：防止因计算误差导致血量出现负数
        safeDamage = min(remaining, target.Health)
        ApplyToHealth(target, safeDamage)
```