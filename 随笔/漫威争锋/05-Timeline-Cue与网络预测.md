# 05 Timeline、Cue 与网络预测

## 1. Timeline 为什么不能等同于 Montage

讲座明确区分：

- Timeline：逻辑流程和时间节点。
- Montage：动画播放和骨骼混合。

当前解包的 `AnimTimeline` 资产也直接证明这一点。奇异博士 `Timeline_101861_Start` 包含：

- `AnimNotifyTracks`
- `AnimSections`
- `SequenceLength = 1.6333333`
- `MontagePath`

而 AbilityItem 再通过 `TimelineToPlayMontage` 把 Timeline 映射到 `AnimResIndex`。这允许皮肤或形态替换动画资源，而逻辑事件不变。

## 2. Timeline 配置模型

### 2.1 最小结构

每个 Timeline 至少定义：

```text
SequenceLength
Sections
NotifyTracks
MontagePath 或资源索引
Version
```

标准技能阶段：

```text
Startup（前摇）
  -> Commit / Spawn / Trace（生效点）
  -> Active / Loop（持续）
  -> Recovery（后摇）
  -> End
```

不要强制所有技能都有同名 Section，但项目应有统一的事件语义，如 `Commit`、`Fire`、`TraceBegin`、`TraceEnd`、`Cancelable`、`Recovery`。

### 2.2 Notify 分类

| 类型 | 可预测 | 权威性 | 示例 |
|---|---|---|---|
| Cosmetic | 是 | 本地 | 镜头、音效、一次性 VFX |
| GameplayIntent | 是 | 服务器复核 | 发射请求、位移开始 |
| AuthorityOnly | 否 | 服务器 | 最终伤害、敌方控制、变身确认 |
| Cleanup | 必须两端幂等 | 两端 | 结束检测、移除临时状态 |

在编辑器里给 Notify 标注类别，可以自动阻止设计师把权威伤害放入纯本地轨道。

### 2.3 变速

技能加速/减速时必须维护一个逻辑时间源：

- Timeline 时间；
- Montage 播放率；
- 连段窗口；
- 周期检测；
- 相机与 Cue 参数。

不要让每个 Task 各自乘一次 Rate，否则叠加减速时会产生不同步。网络同步应传“起始服务器时间 + 当前 Rate + 段 ID”，不逐帧复制时间。

### 2.4 取消与跳段

每个 Section 配置：

- 是否可被用户取消；
- 是否可被控制打断；
- 能跳到哪些 Section；
- 是否已经 Commit；
- 退出时要运行哪些 Cleanup。

Montage 被 Blend Out 不等于 Ability 已结束。Task 应监听 Completed、BlendOut、Interrupted、Cancelled，并把它们映射为统一 Ability 结束原因。

## 3. AbilityTask 的使用规则

UE `UAbilityTask` 是与 Ability 绑定的异步任务。源码注释要求：

- 使用 `NewAbilityTask` 创建；
- 在 `OnDestroy` 注销回调；
- Ability 结束时 Task 一并销毁；
- Task 可读取 Ability 的 ActivationPredictionKey。

Marvel 的技能模板应由 Task 组合，而不是把所有逻辑塞进 Ability Tick。

推荐 Task 边界：

- PlayTimelineAndMontage
- WaitSemanticInput
- PerformScopeTrace
- SpawnProjectile
- SpawnSummon
- ApplyMovement/RootMotion
- WaitGameplayEvent/Tag
- ManageCamera

Task 的输入是配置和 Context，输出是标准事件/结果；Task 不应知道具体英雄 ID。

## 4. GameplayCue 与 TypedCue

### 4.1 UE Cue 生命周期

UE Cue 事件分为：

- OnActive：持续 Cue 开始。
- WhileActive：持续状态存在。
- Executed：瞬时执行。
- Removed：持续 Cue 结束。

Ability 可以直接发复制 Cue；GameplayEffect 也可随 ActiveEffect 自动维护 Cue。逻辑只发 Tag 和参数，不直接 Spawn Niagara。

### 4.2 TypedCue 中间层

讲座描述的 TypedCue 位于 GameplayCue 与 Niagara/SkeletalMesh 之间：

```text
Ability / Effect / Entity
  -> GameplayCue Tag + Parameters
  -> TypedCue（挂点、生命周期、参数适配、对象池）
  -> Niagara / Audio / Material / SkeletalMesh
```

TypedCue 应提供：

- Attach 规则和 Socket；
- 跟随 Owner/Target/HitLocation；
- Begin/Loop/End 资源；
- Cue 参数到 Niagara User Parameter 的映射；
- 可见性、距离、质量等级和平台裁剪；
- 对象池与预分配；
- 同一实例的去重 Key；
- 预测确认/拒绝处理。

### 4.3 CueOptTable

解包字段：

- `CueTag`
- `bEnableOptimization`
- `CuePropertySet`
- `bCustomOptimizationContext`
- `Context`
- `Delivery`

这说明优化策略是数据化的。配置建议：

- 持续且高频 Cue 使用距离和可见性裁剪。
- 关键命中反馈只降低复杂度，不完全裁掉。
- 本地拥有者的准星/受击反馈与远端策略分开。
- 优化只能影响表现，不能阻止逻辑 GameplayEvent。

### 4.4 Niagara Data Interface

Data Interface 适合读取：

- 生命、护盾、阵营；
- 规范化状态 Tag；
- 角色关键点和战斗实体信息。

限制：

- 只读；
- 返回渲染需要的稳定快照；
- 不允许 Niagara 直接应用 Buff、伤害或改变 Ability；
- Dedicated Server 不依赖 Niagara 才能完成逻辑。

## 5. 网络预测模型

### 5.1 UE 的 PredictionKey

UE 5.8 `GameplayPrediction.h` 的注释给出完整协议：

1. Client `TryActivateAbility` 生成 PredictionKey，并调用 ServerTryActivateAbility。
2. Client 不等待服务器，立即 ActivateAbility。
3. 预测副作用关联该 Key。
4. Server 决定成功或失败并回传。
5. 失败触发 Rejected delegate，回滚副作用。
6. 成功仍要等待属性复制追上，CaughtUp 后移除预测副本。

PredictionKey 从客户端到服务器复制；服务器回客户端时只发给原始预测客户端。它不是全局事件 ID，也不能用于让其他客户端相信本地命中。

### 5.2 ScopedPredictionWindow

Ability 激活后发生的后续输入，例如蓄力释放、目标确认、连段点击，需要新的预测窗口：

```text
激活 Key A
  -> WaitInputRelease
  -> 创建 ScopedPredictionWindow，生成 Key B（Base=A）
  -> Client 发送 Release + B
  -> Server 在相同作用域使用 B
```

每个逻辑原子动作使用新的 Key。不要让整段 10 秒持续技能只用一个 Key，否则后续副作用无法独立确认或拒绝。

## 6. 什么应该预测

### 6.1 默认预测

- Ability 激活；
- 自身 Cost/CD；
- 本地动画、镜头、输入锁定；
- 可回滚位移；
- 本地 Projectile/Cue 的视觉代理；
- 自身短期状态，且有明确拒绝清理。

### 6.2 默认不预测

- 对敌最终伤害；
- 敌方眩晕、冰冻、抓取；
- 不可逆变身；
- 会影响其他玩家决策的权威召唤物；
- 复杂连锁 Buff 的最终结果。

“不预测伤害”不等于命中没有即时反馈。可以预测枪口、弹道、准星命中候选和轻量 Cue，血量变化等服务器确认。

## 7. 预测副作用清单

每个 LocalPredicted Ability 在评审时填写：

| 副作用 | 关联 Key | 确认时 | 拒绝时 |
|---|---|---|---|
| Cost | Activation Key | 保留服务器结果 | 恢复预测修改 |
| Cooldown | Activation Key | 合并 ActiveGE | 移除预测 GE |
| Montage | Activation Key | 继续/校时 | 停止或淡出 |
| 位移 | Scoped Key | 服务器校正 | 拉回 |
| Projectile proxy | Fire Key | 与权威实体合并 | 淡出/销毁 |
| Loop Cue | 对应阶段 Key | 去重 | Removed |
| 输入阻挡 | Ability 生命周期 | 保留到 End | 立即清理 |

若某个副作用无法填写拒绝路径，就不应预测。

## 8. 复制模式与 ASC 所有权

UE ASC 支持 Full、Mixed、Minimal GameplayEffect 复制模式。选择原则：

- 玩家 ASC：常用 Mixed，Owner 获得完整 Effect，其他客户端主要获得 Tag/Cue。
- 简单 AI/召唤物：Minimal，前提是远端不需要完整 Effect 详情。
- 调试或少量关键实体：Full。

实际模式由 Marvel 业务实现决定，当前资产只显示召唤物存在 `bNotRepActiveGameplayEffects`、`bNotRepPredictionKeyMap` 等优化开关。不要仅依据一个开关推断全项目模式。

OwnerActor 和 AvatarActor：

- 要跨死亡保留 Ability/CD，ASC 放 PlayerState 等稳定 Owner。
- Avatar 是当前 Pawn/形态。
- 换形态后更新 ActorInfo，并重新检查依赖 Avatar 的 Task、Cue 和输入。

## 9. Cue 与预测去重

预测 Cue 和服务器 Cue 可能同时到达。去重 Key 至少包括：

```text
PredictionKey
CueTag
Target
Effect/Ability Handle
EventType
```

确认时不要再播一遍 OnActive；拒绝时为已经启动的持续 Cue发送 Removed。一次性短 Cue 可允许播放完，但不能残留循环音效或材质。

## 10. 调试与验收

测试矩阵至少覆盖 10 ms、100 ms、200 ms RTT，0%、1%、5% 丢包：

- 正常激活；
- 激活瞬间被控制；
- CD/Cost 客户端可用但服务器不可用；
- 蓄力释放与取消；
- Timeline 变速；
- 预测 Projectile 合并；
- 位移拒绝回滚；
- Cue 去重；
- Ability End 清理；
- 换形态/重生时 ActorInfo 更新。

- [ ] 每个预测阶段有唯一 Key。
- [ ] 所有预测副作用可确认或拒绝。
- [ ] Cue 不双播、不残留。
- [ ] 服务器拒绝后 Tag、Cost、CD、输入、相机均恢复。
- [ ] Dedicated Server 不依赖表现资源。

