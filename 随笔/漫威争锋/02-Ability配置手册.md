# 02 Ability 配置手册

## 1. 配置目标

一个可上线的技能必须同时回答六个问题：

1. 它是谁：稳定 AbilityID、英雄归属、资产入口。
2. 它何时能开始：输入、CD、Cost、必需/阻挡 Tag、优先级。
3. 它开始后拥有什么状态：ActivationOwnedTags、输入阻挡、镜头和 UI 状态。
4. 它怎样执行：GameplayAbility 类、Task、Timeline、检测或实体。
5. 它如何结束：正常结束、取消、死亡、输入释放、超时、预测拒绝。
6. 它怎样被观察：Cue、统计、日志、UI、本地和远端复制。

任何一项缺失，技能都可能“能播放但不能正确战斗”。

## 2. 第一层：AbilityTable

### 2.1 身份与资产

| 字段                     | 应如何配置                  | 原因                         |
| ---------------------- | ---------------------- | -------------------------- |
| `AbilityID`            | 全局稳定，不复用已发布 ID         | Buff、日志、UI、模式覆盖和网络数据都可能引用它 |
| `HeroName`             | 仅用于可读性，不作为主键           | 本地化和改名不应破坏引用               |
| `AbilityAsset`         | 指向唯一 MarvelAbilityItem | 表只做索引，不承载全部行为              |
| `AbilityUser`          | 区分 Hero、Level、Summon 等 | 授予时选择 Owner/Avatar 与数据域    |
| `bEnable` / `Delivery` | 发布流水线控制                | 禁用内容不应被运行时注册               |
| `bCacheSkill`          | 高频或首帧敏感技能开启            | 以内存换激活时延，避免所有技能无差别常驻       |

### 2.2 输入与 UI

| 字段                      | 配置规则                                                                 |
| ----------------------- | -------------------------------------------------------------------- |
| `ActivateInput`         | 使用语义输入名，如 `MouseLeft`、`LeftShift`、`Passive`、`OnlyInitialize`，不要写硬件键码 |
| `TName` / `Description` | 指向 StringTable；调试文本可以 CultureInvariant，发布文本必须可本地化                    |
| `AbilityUITags`         | `Weapon`、`Normal`、`Ultimate`、`Melee` 等用于 UI 布局，不应参与战斗判定              |
| `PersonDescription`     | 第一人称/特殊视角说明与通用说明分离                                                   |
|                         |                                                                      |

Enhanced Input 负责“哪个物理键产生语义动作”，AbilityTable 负责“哪个技能响应语义输入”。这样键位重映射、手柄、无障碍输入和英雄换形态不会修改技能资产。

### 2.3 CD、充能与持久化

| 字段                     | 配置规则                                                |
| ---------------------- | --------------------------------------------------- |
| `AbilityCD`            | 简单固定 CD；复杂阶段 CD 可由 AbilityItem/GE 接管                |
| `AbilityCDID`          | 共享或外部 CD 组 ID；0 表示没有独立组                             |
| `bSaveCDInPlayerState` | 换 Pawn/形态后仍需保留时开启                                   |
| `bSharedCDInTeam`      | 团队唯一资源或共享机关技能才开启                                    |
| `bKeepCDWhenReborn`    | 重生不应刷新的强技能开启                                        |
| `MaxChargeCount`       | 0 表示不用表层充能；大于 0 时必须同时检查 `ChargeInterval`、`ChargeID` |
| `ChargeAutoRemove`     | 消耗充能后是否自动移除对应状态                                     |

UE 的 `UGameplayAbility::CommitAbility` 把 Cost 与 Cooldown 作为一次提交动作；`CostGameplayEffectClass` 和 `CooldownGameplayEffectClass` 是原生扩展点。表层 CD 是项目数据入口，运行时仍应转化为可查询 Tag/Effect，避免 UI 和服务器各维护一套计时器。

### 2.4 战斗与统计元数据

- `QuoteBuff`：声明激活、弹道、打断和自定义阶段引用的 Buff ID。这里只放引用，不复制 Buff 结构。
- `AbilityLogTags`：标记主攻击、投射物等统计维度。
- `AbilityLog`、`BuffLog`、`ToRecordEffectIDs`：明确记录范围，避免所有事件都进入高频日志。
- `DamageTypeClass`：破坏等级、伤害反馈或特殊规则的策略类。
- `ImpulseLevelInfo` / `ImpulseApplyFilter`：冲量等级与可作用对象，不能由命中特效临时决定。
- `BeHitAnimConfigID`：命中反馈配置索引。

## 3. 第二层：MarvelAbilityItem

AbilityItem 是策划的一站式入口。全量解包中出现的字段可分为八组。

### 3.1 授予与特殊配置

| 字段                      | 用法                      |
| ----------------------- | ----------------------- |
| `GrantedAbility`        | 必填，指向 GameplayAbility 类 |
| `GrantedAbilityDataSet` | 技能所需额外数据集               |
| `ExtendConfigInline`    | 技能专有配置对象；只放该模板真正需要的参数   |
| `ActivateData`          | 通用激活参数                  |

`ExtendConfigInline` 应使用 `Instanced`/内联对象语义，跟随 AbilityItem 生命周期。不要把英雄专属字段不断加入 MarvelAbilityItem 主类；当前解包已有至少 157 种 `Config_*`，说明多态扩展是预期设计。

### 3.2 输入绑定

- `BindInputs` / `BindInputConfigs`：激活后的次级输入，如传送门放置、升降、确认、取消。
- `ConfirmInput` / `CancelInput`：目标选择阶段的确认与取消。
- `BlockInputBinds` / `KeyBoardBlockInputBinds`：技能期间阻挡的输入集合。
- `KeyConsumptionPriority`：多个技能监听同一输入时的消费顺序。
- `StandByActivateInput`：待机态激活输入。

每个 BindInput 至少要配置：

```text
MethodName
InputBind
SkillActivationMode（Click/Hold 等）
bTriggerOnRelease
MaxHoldTime / AutoTriggerTime
bBlockAbilityInput
DelayTriggerTime
```

规则：输入只发意图，最终状态转换仍由 Ability 处理；不要在输入回调里直接改属性或生成伤害。

### 3.3 资源消耗

- `PrimaryCost`：主要资源类型。
- `ActivateCostValue`、`CheckActivateCostValue`、`MinimalActivateCostValue`：激活、检查和最低阈值。
- `CostAttrActions`：多属性消耗动作。
- `bCanActiveWithoutEnoughEnergy` / `bEndWithoutEnoughEnergy`：允许启动但资源耗尽结束的持续技能。
- `bInitActiveCost`、`bRestartRegenWhernCost`、`RecoverDelayTime`：资源恢复时序。
- `CostRequireTag` / `CostIgnoreTag`：状态化消耗例外。

检查与提交必须分离。`CanActivateAbility` 阶段只能判断；真正扣费应在 Commit 或可预测的原子阶段执行，否则服务器拒绝时无法统一回滚。

### 3.4 Timeline 与动画映射

`TimelineToPlayMontage` 保存 `Timeline -> AnimResIndex`。一个技能可有多个 Timeline，如起手、循环、结束、不同连段。

配置步骤：

1. Timeline 定义逻辑段和关键事件。
2. AnimResIndex 解析到当前皮肤/形态的 Montage。
3. AbilityTask 同时启动 Timeline 与 Montage。
4. 逻辑命中只监听 Timeline 事件，不能依赖某个皮肤 Montage 的 Notify 名。
5. 播放速率变化必须同时作用于逻辑时间和动画时间。

### 3.5 镜头与震屏

相关字段包括 `CameraInterpAsset`、`AutoActiveCameraInterp`、`AbilityCameraShakeClass`、`HitCameraShakeClass`、`AcivateAbilityShake`、`EnableCameraShake`、`AutoShakeHUD`。

配置原则：

- 本地拥有者镜头与命中目标反馈分开。
- 镜头资产是表现，不参与服务器判定。
- 远端不复制相机对象，只复制足以触发表现的事件。
- 连续技能必须定义退出路径，EndAbility 时恢复相机控制权。

### 3.6 组合技与持续技

- `AbilityComboSections`、`AbilityComboRanges`：连段窗口。
- `bUsingDifferentComboRange`：不同段使用独立输入窗口。
- `CancelLoopAbilityActions`、`CustomActivateInterval`：持续循环与退出动作。
- `GatherCallbackValues`、`AutoBeginGather`：蓄力/采集型技能。
- `bStopLoopActivateWhenUserCancel`：用户取消后停止自动再激活。

连段窗口属于逻辑 Timeline，不等于 Montage Section。Montage Section 可做动画跳转，但是否允许输入、是否消耗资源、是否能被打断必须由 Ability 状态决定。

### 3.7 UI 高亮与瞄准

`HighlightComboScopes`、`ScopeIDNeedHighlight`、`bOnlyHighlightWhenActivated` 等字段只负责提示。真正的目标合法性必须由 Scope 的 TraceRequirements、阵营、遮挡和服务器校验决定。

### 3.8 死亡与中断

- `bCommitCooldownOnDeathInterrupt`：死亡打断时是否仍结算 CD。
- `bEndWithoutEnoughEnergy`：资源不足结束。
- `CancelInputDelayTime`：防误触窗口。
- `bDisableAudioOnFail*`：失败反馈去重。

每个 AbilityItem 都要在评审时写清以下结束矩阵：

| 原因 | 是否 Cancel | 是否 Commit CD | 是否退还 Cost | 是否清理实体 | 是否播结束 Cue |
|---|---|---|---|---|---|
| 正常完成 | 否 | 是 | 否 | 是 | 是 |
| 用户取消 | 是 | 设计决定 | 设计决定 | 是 | 是 |
| 被控制打断 | 是 | 通常是 | 通常否 | 是 | 是 |
| 死亡 | 是 | 看字段 | 否 | 是 | 可抑制 |
| 预测拒绝 | 是 | 回滚预测提交 | 回滚预测提交 | 回滚可回滚实体 | 去重 |

## 4. 第三层：GameplayAbility 蓝图/CDO

### 4.1 标签身份

每个技能至少配置：

```text
Ability.ID.<AbilityID>
Ability.Type.<大类>
Ability.Log.<统计分类>（如需要）
```

不要只用 ID Tag。关系规则应该尽量作用于分类 Tag，例如“控制状态阻挡所有 `Ability.Type.Status.Move`”，而不是列举几十个技能 ID。

### 4.2 激活条件

| UE 字段 | 含义 |
|---|---|
| `ActivationRequiredTags` | Owner 必须具备全部 Tag |
| `ActivationBlockedTags` | Owner 具备任意阻挡 Tag 时不能激活 |
| `ActivationOwnedTags` | Ability 激活期间临时授予 Owner |
| `AbilityTriggers` | 由 GameplayEvent 或 Owner Tag 变化触发 |

激活失败原因应能映射到 UI/音频：CD、Cost、Blocked、MissingRequired、用户逻辑。MarvelAbilityItem 中对应多组 `bDisableAudioOnFail*`，说明失败反馈已经按原因分类。

### 4.3 激活时的取消与阻挡

| 字段 | 执行动作 |
|---|---|
| `CancelAbilitiesWithTag` | 激活时取消匹配的已激活技能 |
| `BlockAbilitiesWithTag` | 激活期间阻止匹配技能开始 |
| `CanceledByAbilitiesWithTag` | Marvel 反向关系描述 |
| `CancelWithOwnerTagAdd/Remove` | Owner Tag 变化时结束当前技能 |

奇异博士 `Ability_101861_BP` 会取消多个普通状态 Ability，并阻挡飞行、其他技能和交互。这不是“传送门逻辑的一部分”，而是关系层配置。

### 4.4 Instancing 与网络策略

UE 原生关键字段：

- `InstancingPolicy`：NonInstanced、InstancedPerActor、InstancedPerExecution。
- `NetExecutionPolicy`：LocalPredicted、LocalOnly、ServerInitiated、ServerOnly。
- `ReplicationPolicy`：Ability 实例状态是否复制。
- `bReplicateInputDirectly`：通常不建议依赖，优先用通用复制事件/Task。
- `bServerRespectsRemoteAbilityCancellation`：客户端取消是否被服务器接受。

建议：

| 技能类型 | Instancing | NetExecution |
|---|---|---|
| 普通输入技能 | PerActor | LocalPredicted |
| 无状态被动 | NonInstanced 或 PerActor | ServerInitiated/ServerOnly |
| 可并发多实例投放 | PerExecution | 取决于回滚能力 |
| 纯本地 UI/镜头 | PerActor | LocalOnly |

任何包含成员变量、异步 Task 或动态委托的 Ability 都不应使用 NonInstanced。

## 5. Ability 生命周期标准模板

```text
Grant
  -> InitAbilityActorInfo
  -> Input / Event
  -> TryActivateAbility
      -> 标签、CD、Cost、业务条件检查
      -> 建立 PredictionKey（LocalPredicted）
  -> ActivateAbility
      -> CommitAbility（一次且只有一次）
      -> 添加 ActivationOwnedTags
      -> 启动 Task / Timeline / Montage / Cue
      -> 命中后通过数据管线应用 Buff/Effect
  -> EndAbility 或 CancelAbility
      -> 停止 Task、Timeline、Cue、镜头
      -> 清理输入绑定、召唤物引用和委托
      -> 移除激活期 Tag
```

UE 头文件明确要求 `ActivateAbility` 路径调用 `CommitAbility`，并最终调用 `EndAbility`。把技能做成“事件跑完但不 End”的蓝图会残留阻挡 Tag、Task 和预测状态。

## 6. Ability 配置验收表

- [ ] AbilityID 唯一，表行、ID Tag、目录名一致。
- [ ] AbilityAsset、GrantedAbility、ExtendConfigInline 引用有效。
- [ ] 输入来自语义 Action，不硬编码物理键。
- [ ] CD/Cost 只在 Commit 阶段真正扣除。
- [ ] Required、Blocked、Owned、Cancel、Block 五类 Tag 关系完整。
- [ ] InstancingPolicy 与成员状态匹配。
- [ ] NetExecutionPolicy 与可回滚性匹配。
- [ ] Timeline 和 Montage 播放速率同步。
- [ ] 正常、取消、死亡、资源耗尽、预测拒绝都有结束路径。
- [ ] EndAbility 清理 Task、Cue、相机、输入、委托和实体句柄。
- [ ] UI 高亮不作为命中合法性依据。
- [ ] 玩法覆盖不直接修改基础资产。

