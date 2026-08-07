# 《漫威争锋》AbilitySystem 配置研究

> 基于讲座笔记、FModel 解包 JSON 与 UE 5.8 GameplayAbilities 源码头文件的交叉研究。

## 文档定位

这套文档回答的不是“UE GAS 的 API 怎么调用”，而是一个更具体的问题：如果要复现《漫威争锋》这种规模的战斗框架，Ability、Effect、Buff、Timeline、Cue、检测、弹道、召唤物、英雄基础数据和玩法覆盖数据应如何分层配置，以及 UE 为什么支持这种分法。

研究对象：

- 讲座原文：`漫威争锋GAS架构讲座.md`
- 解包根目录：`D:\FModel\Output\Exports\Marvel\Content\Marvel`
- UE 5.8 GAS：`E:\UEProject\UE_5.8\Engine\Plugins\Runtime\GameplayAbilities`
- 解包规模：11,460 个 JSON；其中 `AbilitySystem` 7,811 个，`Data` 3,649 个
- 英雄基础表：截至本次解包包含 1011 至 1065、训练机器人及若干模式角色

## 阅读顺序

1. [01-架构总览与证据边界](01-架构总览与证据边界.md)
2. [02-Ability配置手册](02-Ability配置手册.md)
3. [03-Effect-Buff-CDR配置手册](03-Effect-Buff-CDR配置手册.md)
4. [04-战斗实体与三大管线](04-战斗实体与三大管线.md)
5. [05-Timeline-Cue与网络预测](05-Timeline-Cue与网络预测.md)
6. [06-全英雄配置图谱](06-全英雄配置图谱.md)
7. [07-UE5.8源码依据与落地检查表](07-UE5.8源码依据与落地检查表.md)
8. [附录A-字段字典与资产索引](附录A-字段字典与资产索引.md)

## 一句话架构

```text
输入 / 玩法规则
    -> AbilityTable（技能注册、UI、CD、资产入口）
    -> MarvelAbilityItem（通用技能配置）
    -> GameplayAbility 蓝图（生命周期、标签关系、网络策略）
    -> ExtendConfigInline（英雄技能特有参数）
    -> Timeline / Task（时序行为）
    -> Scope / Projectile / InstantAmmo / Summoner（检测与实体）
    -> BuffSpec（一次业务状态实例）
    -> Effect_NNNN（可复用 GameplayEffect 模板）
    -> GameplayCue / TypedCue（表现）
```

## 结论先行

- 不要做“一张万能技能表”。稳定的通用字段、技能私有字段、技能间关系必须分开。
- Ability 负责“能否开始、怎样进行、怎样结束”；命中和伤害不应直接塞进 Ability 表。
- Effect 是少量、可复用的状态算法模板；BuffSpec 是大量、易迭代的业务实例。
- `EffectID + AttributeValue + Duration + Tags + Cue` 是《漫威争锋》Buff 数据化的核心组合。
- 检测源和被检测目标都必须能配置规则，否则英雄数量增长后会形成 M×N 硬编码。
- Timeline 是逻辑时序，不是 Montage 的替代品；二者通过映射保持逻辑与动画解耦。
- 预测只覆盖影响输入手感且可回滚的动作；伤害、控制和复杂变身默认由服务器裁决。
- 玩法差异应通过表合并、重定向和效果器覆盖实现，不复制整套英雄资产。

## 证据标记

文档按章节集中说明证据边界；需要标记单项结论时使用三种标记：

- **[资产]**：可在当前解包 JSON 中直接验证。
- **[源码]**：可在 UE 5.8 GameplayAbilities 头文件中直接验证。
- **[推导]**：讲座概念与资产结构吻合，但 Marvel 业务 C++ 未包含在工作区，具体执行顺序属于高可信推导。

不能从解包资产反推出的内容会明确写成“未知”，不会用 UE 原生行为冒充 Marvel 私有实现。
