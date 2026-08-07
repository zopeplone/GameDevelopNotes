# 03 Effect、Buff 与 CDR 配置手册

## 1. 三层模型

讲座给出的三层模型可以用当前资产精确解释：

```text
Effect：少量、复用、偏算法
    定义属性修改、Tag、免疫、互斥、叠层、GEComponent

BuffSpec：大量、业务化、偏实例
    引用 EffectID，填数值、时长、来源生命周期、Cue、链式 Buff

CDR：最细粒度的 Condition / Disable / Restrict
    通过 Granted Tags、Tag Requirements、Block/Cancel 关系被各系统消费
```

CDR 在当前导出中不是一张名为 CDR 的表，而是以 Tag 和各系统条件字段分布存在。因而 CDR 应理解为“状态协议层”，不是必须对应一个 UObject 类。

## 2. Effect 模板层

### 2.1 资产事实

`AbilitySystem/Common/GameEffect` 中有 211 个 `Effect_*` 资产，`Common/QA` 另有 1 个测试资产。典型资产继承 `MarvelGameplayEffectTemplate`，并组合 UE 5.8 的 GameplayEffectComponent：

- AssetTagsGameplayEffectComponent
- TargetTagsGameplayEffectComponent
- TargetTagRequirementsGameplayEffectComponent
- ImmunityGameplayEffectComponent
- RemoveOtherGameplayEffectComponent
- AbilitiesGameplayEffectComponent
- ChanceToApplyGameplayEffectComponent
- Marvel 自定义的 ShapeShift、GravityZone、SourceEffectChance 等组件

UE 5.8 把许多旧版 GameplayEffect 逻辑迁入 `GEComponents`，所以新项目也应优先组合组件，而不是继续膨胀一个 GameplayEffect 子类。

### 2.2 Effect 应配置什么

#### 身份与分类 Tag

每个 Effect 至少有：

```text
Effect.<EffectID>
CustomEffect.Type.<分类>
```

可选分类示例：

- `NegativeEffect`：负面状态。
- `NumericalBuff`：数值增益。
- `OffensiveAssist` / `DefensiveAssist`：战斗统计归因。
- Audio/VFX 分类 Tag：供表现层查询。

Asset Tag 表达“这个 Effect 是什么”；Owned Tag 表达“目标当前处于什么状态”。两者不能混用。

#### DurationPolicy 与 DurationMagnitude

| 类型 | 用法 |
|---|---|
| Instant | 一次执行，不保留 ActiveGameplayEffect |
| Infinite | 无限持续，直到显式移除 |
| HasDuration | 有限时长，可由常量、属性、MMC 或 SetByCaller 计算 |

Marvel 模板常把 DurationMagnitude 配成 SetByCaller，Tag 为 `CustomEffect.Duration`。这使一个 Effect 模板能被不同 BuffSpec 用不同持续时间复用。

规则：

- BuffSpec 决定业务时长时，Effect 用 SetByCaller。
- Effect 的语义天然固定时才写常量。
- Instant Effect 不要配置 Period。
- 无限状态必须有清晰的移除所有者，如 Ability、来源 Actor、死亡或显式 Handle。

#### Modifiers

每个 Modifier 配置：

1. Attribute：如 MoveSpeed、Damage、Health。
2. ModifierOp：Additive、Multiplicitive、Division、Override。
3. Magnitude：ScalableFloat、AttributeBased、CustomCalculation、SetByCaller。
4. EvaluationChannel：运算通道。
5. Source/Target Tag Requirements：该 Modifier 局部生效条件。

`Effect_1116` 的 MoveSpeed Modifier 使用 SetByCaller `Hero.AttributeData`，BuffSpec 只需给出 `AttributeValue`。这种模式避免为 +10%、+20%、+600 各建一个 Effect。

运算建议：

- 固定值增减用 Additive。
- 百分比倍率用 Multiplicitive，并统一“0.2 是 +20%”还是“1.2 是最终倍率”的项目约定。
- 硬覆盖只用于明确的状态，如速度锁定；必须定义优先级。
- 复杂伤害公式用 ExecutionCalculation，而不是堆多个顺序敏感 Modifier。

#### Executions

Execution 适合：

- 伤害结算；
- 护盾、护甲、减伤、暴击等多属性交叉；
- 需要 Source/Target 捕获属性的治疗；
- 需要一次产生多个 OutputModifier 的公式。

持续属性 Buff 不应滥用 Execution，因为普通 Modifier 能获得更好的聚合、回滚和调试行为。

#### Application / Ongoing / Removal Requirements

三类 Requirement 职责不同：

- Application：能否首次应用。
- Ongoing：Active Effect 是否暂时被抑制。
- Removal：目标满足条件时移除。

如果只想“免疫某类控制”，优先配置 GameplayEffectQuery/ImmunityComponent；不要让每个 Buff 蓝图写一次 `HasTag`。

#### Cancel、Block 与 Owned Tags

控制类 Effect 应通过组件和 Tag 关系完成：

- TargetTags：给目标添加状态 Tag。
- CancelAbilityTags：应用时取消技能。
- BlockAbilityTags：持续期间阻挡技能。
- RemoveOtherEffect：移除互斥或低优先级状态。

例如 `Effect_1109`：

- Owned Tags 包含禁止相机转向、受击硬直和 AI 寻路的状态；
- Cancel 列表覆盖交互、飞行、移动、主技能等类型；
- Block 列表还包含可打断技能类型；
- RemoveGameplayEffectQuery 清理若干互斥 Effect；
- StackingType 为 Replace。

这说明“眩晕/静滞”不是一个布尔值，而是由一个 Effect 原子地安装一组能力限制。

### 2.3 Stacking

配置时必须同时决定五件事：

| 决策 | UE/Marvel 字段 |
|---|---|
| 按来源还是按目标聚合 | `StackingType` / `StacksRule` |
| 最大层数 | `StackLimitCount` |
| 达上限后是否拒绝 | `bDenyOverflowApplication` / `DenyOverflow` |
| 再次施加是否刷新时长 | `StackDurationRefreshPolicy` / `RefreshStackTime` |
| 周期计时是否重置 | `StackPeriodResetPolicy` |

推荐语义：

- 同一来源 DoT：AggregateBySource，便于不同攻击者分别计时。
- 团队共享减速：AggregateByTarget 或 Replace，避免来源数量改变强度。
- 不可叠控制：Replace，并明确是否刷新时长。
- 层数增益：设置上限，并决定溢出时触发额外效果还是拒绝。

不要只改 `StackLimitCount`。叠层行为是多个字段共同定义的状态机。

### 2.4 GameplayCue

Effect Cue 配置应遵循生命周期：

- Instant/Execution：Executed。
- Duration/Infinite：OnActive、WhileActive、Removed。
- 叠层时用 `bSuppressStackingCues` 控制是否每层重复触发。
- 业务实例需要换 Cue 时，由 BuffSpec 的 `bOverrideEffectCues` 覆盖。

## 3. BuffSpec 实例层

### 3.1 核心字段

| 字段 | 含义与配置原则 |
|---|---|
| `ID` | Buff 全局稳定 ID |
| `EffectID` | 引用通用 Effect 模板 |
| `AttributeValue` | 写入 Effect 的 SetByCaller 数值 |
| `Duration` | `-1` 常表示无限；需与生命周期字段一致 |
| `Period` | 周期，不使用时为 0 |
| `BuffAttributeType` | 普通/特殊属性解释策略 |
| `ExtendedConfig` | Buff 私有业务配置 |
| `Desc` | 策划说明，不作为逻辑 |

典型组合：

```text
BuffSpec 10340008（钢铁侠玩法词条）
  EffectID = 1116（MoveSpeed 模板）
  AttributeValue = 600
  Duration = -1
  GrantTags = CustomEffect.UnlimitMoveSpeed
  StacksRule = NoStack
```

### 3.2 生命周期 LifeWith

`LifeWith` 是 Marvel 对 UE ActiveGameplayEffect 生命周期的业务封装：

- `LifeWithAbility`：Ability 结束时移除。
- `LifeWithAbilityDestroy`：Ability 实例销毁时移除。
- `LifeWithAlive` / `LifeWithTargetAlive`：角色死亡时移除。
- `LifeWithCauser`：Causer 销毁时移除。
- `LifeWithInstigator`：Instigator 失效时移除。
- `LifeWithModifierClean`：净化/Modifier 清理时移除。

配置规则：每个无限 Buff 至少选择一个明确的生命所有者。多个标志同时开启时应按“任一条件失效即移除”实现，并在编辑器中展示实际移除条件。

`bOwnerDeathAutoRemove` 与 LifeWithAlive 有重叠时，要在数据校验器中给出警告，避免设计师误以为二者是叠加时长。

### 3.3 百分比伤害与治疗

- `PercentDamageHealthType` + `PercentDamagePercent`：按当前/最大/已损生命等口径造成百分比伤害。
- `PercentTreatType` + `PercentTreatPercent`：百分比治疗。

这些字段应转换为统一 Execution 输入。不要在 BuffSpec 里直接改 Health，否则会绕过护盾、伤害归因、死亡和预测规则。

### 3.4 叠层覆盖

BuffSpec 有 `bOverrideEffectStackRule`。只有业务实例必须改变模板语义时才开启，并填写：

- `StacksRule`
- `StackLimitCount`
- `DenyOverflow`
- `StackingExpirationPolicy`
- `StackPeriodResetPolicy`
- `RefreshStackTime`
- `bRefreshWhenReplace`
- `bAccumulateDuration` / `MaxAccumulateDuration`

模板默认值应覆盖 80% 的实例。大量 Buff 都打开 Override 说明 Effect 划分过粗，应拆出新的通用模板。

### 3.5 预测字段

- `bShouldPredictSpawn`：预测生成本地 Buff 表现/状态。
- `bShouldPredictDelete`：预测移除。
- `bInvalidSyncBuff`：不参与通常同步的业务标记。
- `SyncRadius`：范围同步或相关性控制。

只对以下 Buff 预测：

- 由本地输入立即产生；
- 只作用于自己；
- 可通过 PredictionKey 移除；
- 被拒绝时恢复成本低。

敌方控制、最终伤害、复杂变身默认不预测。

### 3.6 GrantedAbilities

Buff 可临时授予 Ability：

```text
Ability class
ActivateOnceImmediately
AbilityID
RemovalPolicy
```

RemovalPolicy 必须明确：Buff 消失时立即取消、允许自然结束，还是仅移除 Spec。持续控制逻辑常以“授予一次并立即激活的辅助 Ability”实现；这比让 GE 自己 Tick 复杂行为更符合 GAS 的职责分离。

### 3.7 Tag、阻挡与取消

| 字段 | 作用 |
|---|---|
| `AssetTags` | Buff 自身分类 |
| `GrantTags` | 应用到目标的状态协议 |
| `BlockAbilityTags` | 持续阻挡 Ability |
| `CancelAbilityTags` | 应用时取消 Ability |

Buff 层字段是对 Effect 模板的业务补充。若大量 Buff 重复相同 Tag 集合，应回收进 Effect 模板。

### 3.8 Cue 与链式 Buff

- `GameplayCues`：实例 Cue。
- `bOverrideEffectCues`：替换模板 Cue。
- `bSuppressStackingCues`：叠层去重。
- `BgnBuffIds`：开始时追加 Buff。
- `EndBuffIds`：结束时追加 Buff。
- `BreakBuffIds`：被打断时追加 Buff。
- `CustomBuffIds`：业务阶段引用。

链式 Buff 必须做环检测。至少在 Cook 前验证：

- A 的 Begin 不能最终回到 A；
- End/Break 不应同时生成互相替换的无限 Buff；
- 链深有上限；
- 每次链式应用保留原始 Source、Instigator 和 PredictionKey。

## 4. CDR 状态协议层

### 4.1 定义

CDR 是 Condition、Disable、Restrict 的集合：

- Condition：某行为成立的前置条件。
- Disable：直接禁止某行为。
- Restrict：允许行为但限制参数或范围。

### 4.2 推荐 Tag 命名

```text
State.CrowdControl.Stun
State.Movement.Rooted
State.Targeting.Untargetable
State.Damage.Invulnerable
Ability.Event.Block_Input
Ability.Event.Block_CameraTurn
CustomEffect.DisableAIPathFollowing
```

协议要求：

- Tag 名描述事实，不描述来源英雄。
- `Hero.1038.SkillX.Stun` 不应成为全局消费条件。
- 每个 Tag 建立 Owner：谁添加、谁移除、哪些系统消费。
- 同一语义只保留一个规范 Tag，旧 Tag 用 Redirect 迁移。

### 4.3 CDR 的消费位置

- Ability 激活条件：Required/Blocked。
- Ability 激活关系：Cancel/Block。
- Scope：TraceRequirements、CollisionTagRequirements、目标过滤。
- Projectile：碰撞、反弹、穿透、阵营变更规则。
- Movement：速度上限、跳跃、飞行、转向。
- AI：寻路与感知。
- Cue/Niagara：状态表现。

这就是为什么 CDR 不应是一堆只被 BuffSystem 理解的枚举；它必须是全战斗框架共享的 Tag 协议。

## 5. Effect 与 Buff 的创建流程

1. 先定义玩法语义：属性、状态、持续、来源、叠层、互斥、预测。
2. 检查已有 Effect 模板能否表达。
3. 能复用：只新增 BuffSpec。
4. 不能复用：新增 Effect 模板或 GEComponent，再新增 BuffSpec。
5. 为 Effect 配置 Asset/Owned Tag、Modifier/Execution、组件和默认叠层。
6. 为 BuffSpec 配 EffectID、SetByCaller 数值、时长和 LifeWith。
7. 配置 CDR Tag 与 Ability/Scope/Movement 的消费关系。
8. 配置 Cue；逻辑不得依赖 Cue 是否成功加载。
9. 测试同源、异源、重复施加、净化、死亡、来源销毁和预测拒绝。

## 6. 常见错误

- 每个数值都建一个 Effect，导致模板爆炸。
- 把技能伤害直接写在 Ability 蓝图，绕过统一结算。
- Infinite Buff 没有生命所有者。
- Asset Tag 和 Owned Tag 混用。
- 只设置最大层数，不设置刷新与周期策略。
- 用 Cue 或 Niagara 回调改变权威战斗状态。
- 敌方控制做本地预测，拒绝时无法恢复。
- Buff 链没有环检测。
- 模式覆盖直接改基础 Effect，污染其他玩法。
