# 核心战斗循环与状态流转 (Combat Loop - MVP Version)

> **设计说明**：本图基于早期手绘草图重构。为了确保项目能顺利开发出第一个可玩版本（Demo 1.0），我们执行了“最小切片（MVP）”原则。
> 
> 当前实装（实线部分）：体术、黑闪、心流（含断连判定）、能量护盾、破盾保护、疲弱期。
> 暂不实装（虚线部分）：咒力压制（弱者悲鸣）、术式熔断、脑伤。这些将作为高级接口保留，待后续版本更新。

```mermaid
flowchart TD
    %% ==== 核心实装循环 (Demo 1.0 目标) ====
    Start([战斗发起]) --> Taijutsu[体术起手/攻击]
    
    %% 1. 攻击与黑闪判定
    Taijutsu --> BF_Check{黑闪判定<br/>PRD保底机制}
    
    %% 2. 未触发黑闪的常态与能量消耗
    BF_Check --> |未触发| Energy_Check{累计咒力消耗<br/>是否达标?}
    Energy_Check --> |否| Taijutsu
    Energy_Check --> |是| Exhaustion[进入疲弱期<br/>护盾恢复减半/受击加重]
    Exhaustion --> Dodge_Counter[闪避 / 反击]
    Dodge_Counter --> Taijutsu

    %% 3. 触发黑闪与心流状态
    BF_Check --> |触发黑闪| Zone[进入心流状态 Zone]
    Zone --> Combo_1_5[连击阶段 1~5<br/>暴击率递增]
    
    %% 心流连击的命中与断连
    Combo_1_5 --> |命中| Combo_6_8[连击阶段 6~8<br/>暴击率极高频]
    Combo_1_5 --> |空挥 / 被打断| Zone_Break[心流中断]
    
    Combo_6_8 --> |全中| Zone_End[心流自然结束]
    Combo_6_8 --> |空挥 / 被打断| Zone_Break
    
    Zone_Break --> Energy_Check
    Zone_End --> Energy_Check

    %% 4. 防御端：护盾受击与破盾保护 (独立事件线)
    Enemy_Attack([敌人攻击命中]) --> Shield_Hit{是否命中护盾?}
    Shield_Hit --> |护盾未破| Taijutsu
    Shield_Hit --> |护盾破碎| Shield_Protection[触发破盾保护<br/>溢出伤害衰减30% + 独立CD]
    Shield_Protection --> Stun[破盾僵直 3s]
    Stun --> Taijutsu

    %% ==========================================
    %% 以下为远期高级接口 (Demo 1.0 暂不实装)
    %% ==========================================
    
    subgraph Advanced[远期高级博弈接口 暂不实装]
        direction TB
        Burnout([术式熔断]) -.-> Brain_Damage[脑伤: 属性大减]
        Brain_Damage -.-> |成功触发黑闪| ClearDebuff[消除脑伤]
        ClearDebuff -.-> Win_Burnout((绝境翻盘))
        Brain_Damage -.-> |双数伤害反转| Lose_Burnout((暴毙))
        
        Suppression([遭遇咒力压制]) -.-> Weak_Cry[弱者悲鸣: 暴击提升/伤害降低]
        Weak_Cry -.-> |强行多次黑闪| Desperate_BF{绝境突围}
        Desperate_BF -.-> |失败| Dead((秒死))
        Desperate_BF -.-> |成功| Breakthrough((突破压制))
    end

    %% 样式定制
    classDef core fill:#1e88e5,stroke:#fff,color:#fff;
    classDef win fill:#2e7d32,stroke:#fff,color:#fff;
    classDef lose fill:#c62828,stroke:#fff,color:#fff;
    classDef advanced fill:#e0e0e0,stroke:#9e9e9e,color:#424242,stroke-dasharray: 5 5;
    
    class Zone,Combo_1_5,Combo_6_8,Zone_Break core;
    class Win_Burnout,Breakthrough win;
    class Lose_Burnout,Dead lose;
    class Burnout,Brain_Damage,ClearDebuff,Suppression,Weak_Cry,Desperate_BF advanced;