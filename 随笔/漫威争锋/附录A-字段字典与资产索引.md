# 附录 A 字段字典与资产索引

## 1. 使用说明

本附录记录当前解包实际出现的字段集合。字段语义由字段名、代表性资产和 UE 类型交叉解释；Marvel 私有 C++ 未包含在工作区的字段只给出配置用途，不承诺内部算法。

## 2. AbilityTable 字段

### 2.1 基础与发布

| 字段 | 用途 |
|---|---|
| AbilityID | 技能主键 |
| HeroName | 策划可读英雄名 |
| TName / Description / PersonDescription | 本地化名称和说明 |
| AbilityAsset | MarvelAbilityItem 引用 |
| AbilityUser | Hero/Level/Summon 等使用者类型 |
| bEnable / Delivery | 启用与交付 |
| bCacheSkill | 是否预缓存 |

### 2.2 输入、CD、充能

| 字段 | 用途 |
|---|---|
| ActivateInput | 语义输入 |
| AbilityCD / AbilityCDID | 固定 CD 和 CD 组 |
| bSaveCDInPlayerState | 跨 Avatar 保留 CD |
| bSharedCDInTeam | 团队共享 CD |
| bKeepCDWhenReborn | 重生保留 |
| bIgnoreSaveEnegy | 资源持久化例外 |
| MaxChargeCount / ChargeInterval / ChargeID | 充能数量、间隔、组 |
| ChargeAutoRemove | 消耗后自动移除充能状态 |

### 2.3 战斗、UI、统计

| 字段 | 用途 |
|---|---|
| QuoteBuff | Active/Missile/Break/Custom Buff 引用集合 |
| AbilityLogTags / AbilityRedirectLog | 统计分类和重定向 |
| AbilityLog / BuffLog / ForbiddenAbilityBaseLog | 事件记录配置 |
| ToRecordEffectIDs | 需要记录的 Effect |
| AbilityStatisticsClass / AbilityExtraStatisticsClass | 统计策略类 |
| DamageTypeClass | 伤害/破坏策略 |
| ImpulseLevelInfo / ImpulseApplyFilter | 冲量等级和对象过滤 |
| BeHitAnimConfigID | 受击反馈配置 |
| AbilityUITags | Weapon/Normal/Ultimate/Melee 等 UI 分类 |
| HeroCopyAbilityParamsMap | 复制其他英雄技能时的参数映射 |

## 3. MarvelAbilityItem 字段组

当前全量数字 AbilityItem 中观察到以下通用字段。

### 3.1 类与配置

`GrantedAbility`、`GrantedAbilityDataSet`、`ExtendConfigInline`、`ActivateData`。

### 3.2 输入

`ActivateInput`、`StandByActivateInput`、`BindInputs`、`BindInputConfigs`、`ConfirmInput`、`CancelInput`、`BlockInputBinds`、`KeyBoardBlockInputBinds`、`KeyConsumptionPriority`、`CancelInputDelayTime`、`ConfirmTipsParam`、`CancelTipsParam`。

### 3.3 Cost/资源

`PrimaryCost`、`ActivateCostValue`、`CheckActivateCostValue`、`MinimalActivateCostValue`、`CostAttrActions`、`CostClipTypeTag`、`CostIgnoreTag`、`CostRequireTag`、`bCanActiveWithoutEnoughEnergy`、`bEndWithoutEnoughEnergy`、`bInitActiveCost`、`bSlackCheckActivateCost`、`SlackCheckCostTime`、`RecoverDelayTime`、`RegenValue`、`ReduceValue`、`InitValue`、`MaxValue`、`DamageRegen`、`TreatRegen`、`bDefaultRegen`、`bRestartRegenWhernCost`、`bFollowIgnoreCost`。

### 3.4 Timeline/连段

`TimelineToPlayMontage`、`AbilityComboSections`、`bChooseCameraShakeByCombo`、`bUsingDifferentComboRange`、`HighlightComboScopes`、`CancelLoopAbilityActions`、`CustomActivateInterval`、`GatherCallbackValues`、`AutoBeginGather`、`bStopLoopActivateWhenUserCancel`。

### 3.5 镜头与反馈

`CameraInterpAsset`、`AutoActiveCameraInterp`、`AbilityCameraShakeClass`、`HitCameraShakeClass`、`HitEnemyCameraShakeClass`、`HitEnemyCameraShakeClassesByCombo`、`AcivateAbilityShake`、`EnableCameraShake`、`AutoShakeHUD`、`ShakeHUDType`、`PPTransparentDistanceOffset`、`AbilityGameplayCues`。

### 3.6 命中高亮

`HitCharactersAndNonShieldSummons`、`HitShieldSummons`、`HitDestructibleScenes`、`HitScenes`、`ScopeIDNeedHighlight`、`bOnlyHighlightWhenActivated`。

### 3.7 阶段 CD 与 Cost

`CommitCooldownStage`、`CommitCooldownStageTag`、`ReadyCooldownStage`、`ReadyCooldownStageTag`、`ReadyCostStage`、`ReadyCostStageTag`、`bCommitCooldownOnDeathInterrupt`。

### 3.8 复制与失败反馈

`bReplicateDataSet`、`bDisableAudioOnFailBlock`、`bDisableAudioOnFailCooldown`、`bDisableAudioOnFailCost`、`bDisableAudioOnOtherCanActivate`、`bDisableAudioOnUserLogic`。

## 4. GameplayAbility CDO 常用字段

解包蓝图 CDO 同时包含 UE 原生字段和 Marvel 扩展字段。

### 4.1 UE 原生核心

- AbilityTags
- AbilityTriggers
- ActivationRequiredTags / ActivationRequiredAnyTags / ActivationBlockedTags
- ActivationOwnedTags / ActivationOwnedTags_AI
- CancelAbilitiesWithTag / BlockAbilitiesWithTag
- CooldownGameplayEffectClass / SkillCostGameplayEffectClass
- NetExecutionPolicy / ReplicationPolicy
- bIsCancelable / bRetriggerInstancedAbility
- bReplicateInputDirectly / bServerRespectsRemoteAbilityCancellation

### 4.2 Marvel 关系和阶段扩展

- AbilityMainTag / AbilityLabels
- ActivationLoopAbilityBlockedTags
- CanceledByAbilitiesWithTag / BlockedByAbilitiesWithTag
- CancelAllowedPolicy / CancelAllowedConfigType
- BreakCancelConfig / BreakStageConfig
- MissileCancelConfig / MissileStageConfig
- PreMissileCancelConfig / PreMissileStageConfig
- ConsistentCancelConfig
- EnterBreakRemoveActivateBlockTags
- SlackActivationRequiredTags / SlackTagTime
- SourceRequiredTags / SourceBlockedTags / Source Tag Requirements
- Target Tag Requirements
- GameplayTagRequirementsToCancelThisAbility

### 4.3 持续、Tick、同步

- bIsDurableAbility
- bContinuouslyActiveAbilityLoopCheck
- bEnableActivationBuffer
- bEnableCommitPrediction
- bEnableMoveLogicFrame
- TickEnableWithAbilityActive
- PrimaryAbilityTick
- LoopAbilityInterval
- MaxForceEndByRemoteTime_Server / MinForceEndByRemoteTime_Server
- bUseSync

蓝图中还存在大量技能私有变量，不能视为通用 schema；它们应迁入 `Config_*` 或模板专属类，避免以字段名反射驱动跨英雄逻辑。

## 5. BuffSpecTable 字段

### 5.1 身份与数值

`ID`、`HeroName`、`EffectID`、`Desc`、`AttributeValue`、`AttributeDiffusion`、`ExtendedConfig`、`BuffAttributeType`、`PercentDamageHealthType`、`PercentDamagePercent`、`PercentTreatType`、`PercentTreatPercent`、`Duration`、`Period`。

### 5.2 生命周期

`LifeWith`、`bCanAppliedWhenDeath`、`bOverrideEffectLifeSpan`、`bModifierCleanAutoRemove`、`bOwnerDeathAutoRemove`、`bTerrainDeathRelevancy`、`KillTerrainTime`。

### 5.3 叠层

`bOverrideEffectStackRule`、`StacksRule`、`DenyOverflow`、`StackingExpirationPolicy`、`StackPeriodResetPolicy`、`RefreshStackTime`、`StackLimitCount`、`bUseRemainingMagnitudes`、`bRefreshWhenReplace`、`bAccumulateDuration`、`MaxAccumulateDuration`、`bBlockApplyFromDifferentSource`。

### 5.4 预测和同步

`bShouldPredictSpawn`、`bShouldPredictDelete`、`bInvalidSyncBuff`、`SyncRadius`、`bDisableCullDistance`。

### 5.5 Ability、Tag、Cue 与链

`GrantedAbilties`、`MAEBuffBlueprintClass`、`AssetTags`、`GrantTags`、`BlockAbilityTags`、`CancelAbilityTags`、`bOverrideEffectCues`、`GameplayCues`、`bSuppressStackingCues`、`BgnBuffIds`、`EndBuffIds`、`BreakBuffIds`、`CustomBuffIds`、`NeedActionLog`、`Delivery`。

## 6. Effect 模板字段

当前 `Effect_*` CDO 出现：

| 领域 | 字段 |
|---|---|
| 应用 | ApplicationTagRequirements、OngoingTagRequirements、RemovalTagRequirements |
| 生命周期 | DurationPolicy、DurationMagnitude、Period、bCanAppliedWhenDeath、bOwnerDeathAutoRemove |
| 数值 | Modifiers、Executions、bMulitiToAddition |
| 标签关系 | InheritableGameplayEffectTags、InheritableOwnedTagsContainer、CancelAbilitiesWithTag、BlockAbilitiesWithTag |
| 免疫/移除 | GrantedApplicationImmunityQuery、RemoveGameplayEffectQuery |
| 叠层 | StackingType、StackLimitCount、StackDurationRefreshPolicy、Coexist、bDenyOverflowApplication |
| 表现 | GameplayCues、bEnableCullDistance |
| 扩展 | GEComponents、EffectPriority、bModifierCleanAutoRemove |

## 7. ScopeAgentTable 字段

### 7.1 身份/资产

`Id`、`HeroName`、`Desc`、`ScopeAgent`、`AssetTags`、`ExtendedConfig`、`Delivery`、`bOpenCache`。

### 7.2 形状/空间

`TraceType`、`TraceMode`、`TraceRule`、`SceneTraceType`、`Radius`、`Length`、`HalfHeight`、`BoxSize`、`HorizontalAngleDegree`、`VerticalAngleDegree`、`bSectorPercentage`、`SectorStartPercentage`、`SectorEndPercentage`、`SectorRotationOffset`、`AnchorData`、`ShapeOffset`、`RotationType`、`CheckDir`、`AdvanceParams`、`AimDirType`。

### 7.3 目标规则

`TraceRequirements`、`BuffRequirements`、`ExtendFilter`、`UniqueApplyRules`、`ShapeParams_Enemy`、`ShapeParams_Teammate`、`ShapeParams_Scene`、`bUseSpecialTraceShape_Scene`、`bUseSpecialTraceShape_Teammate`、`ResultSortMode`、`MaxApplyTimes`、`MaxTraceNumForOptimization`。

### 7.4 遮挡/传送门

`Occlusion`、`bCanCrossPortal`、`bPortalAsTraceResult`、`bFixTraceEndWhenOcclusion`、`bCloseEvaluateTraceStart`、`bShouldFixTraceHit`、`bTraceSupplement`。

### 7.5 结果

`BaseDamage`、`BuffInfos`、`BuffSpecs_ToSelf`、`RemovalBuffIDs`、`GameplayCues`、`HitImpactCuePlayRule`、`ImpulseApplyFilter`、`ImpulseDirConfig`、`ImpulseDirectionFixAngle`、`ImpulseLevelInfo`、`ScopeQuote`、`ChaosFieldQuote`。

### 7.6 网络/预测

`bUseClientCameraInfo`、`bServerVerifyClientResult`、`bServerWaitLocationInfo`、`bPredictionDelayDamage`、`bEnableRemoteClientCue`、`bShowSync`、`bUserConfirmInput`、`bCommitCostPerTrace`、`bEndScopeWhenCostFailed`、`bEndScopeWhenReachMaxApplyLimitation`。

## 8. ProjectileAgentTable 字段

### 8.1 身份、运动、碰撞

`Id`、`HeroName`、`Desc`、`ProjectileAgent`、`AdvanceParams`、`AimDirType`、`AssetTags`、`ExtendedConfig`、`CollisionRule`、`CollisionTagRequirements`、`bAlternateMovement`、`bCheckProjectileTrajectory`、`bEnablePositionAdjustment`、`bRequireBounceCheckHitObjectState`。

### 8.2 伤害、Buff、引用

`BaseDamage`、`BuffRequirements`、`BuffSpecs`、`BuffSpecs_ToSelf`、`RemovalBuffIDs`、`GameplayCues`、`ScopeQuote`、`SummonerQuote`、`ChaosFieldQuote`、`ImpulseApplyFilter`、`ImpulseLevelInfo`、`BeHitAnimConfigID`。

### 8.3 反弹/穿透

`bShouldBounce`、`BounceRule`、`Bounciness`、`Friction`、`BoundCountMax`、`bApplyDamageWhenBounce`、`bShouldPenetrate`、`PenetrateRule`、`PenetrateCountMax`、`PenetrateDamageFalloffCurve`、`PenetrateDamageFalloffFloor`、`MaxFlyDistanceAfterPenetrate`、`bApplyDamageWhenPenetrate`、`bApplyQuoteWhenPenetrate`。

### 8.4 网络与优化

`bUseClientResult`、`bUseClientPenetrate`、`bUseTraceVerify`、`bServerWaitLocationInfo`、`bNeedAbilityLocalPredicted`、`bBatchFireRPC`、`bBatchPenetrateRPC`、`BatchPenetrateRPCInterval`、`bSwiftProjectile`、`SwiftProjectileType`、`SwiftProjectileHitType`、`bUseVirtualProjectile`、`VirtualProjectile`、`bUseSwiftServerContainer`、`bUsingParallelOptimizationMovement`、`PreCreateCount`、`bOpenCache`、`UpdateRateWhenNotRendered`、`VisibilityBasedUpdateType`、`bDisableActorReplicationToClients`、`bPreferOwnerOnlySync`、`bShowSync`。

## 9. InstantAmmoTable 字段

`AmmoID`、`AmmoClass`、`AmmoFireRange`、`AmmoTraceRule`、`AimDirType`、`AssetTags`、`BaseDamage`、`DestructibleDamage`、`RadiusForAlly`、`RadiusForEnemy`、`RadiusForScene`、`bEnableRadiusForScene`、`TraceRequirements`、`bShouldPenetrate`、`PenetrateCount`、`PenetrateDistance`、`PenetrateRule`、`PenetrateDamageCurve`、`PenetrateDamageExtraAddend`、`PenetrateAimFinalRadiusExpand`、`bDoNotPenetrateSceneObjectOnServer`、`BuffSpecs`、`BuffSpecs_ToSelf`、`GameplayCues`、`ChaosFieldQuote`、`ScopeQuote_Effective`、`ImpulseApplyFilter`、`ImpulseLevelInfo`、`BeHitAnimConfigID`、`Delivery`。

## 10. SummonerAgentTable 字段

### 10.1 实体/属性/数量

`Id`、`HeroName`、`Desc`、`SummonedName`、`SummonerAgent`、`AttrSpec`、`bNeedASC`、`bNeedBuff`、`InitBuffIDs`、`MaxNum`、`QuantityGroupID`、`SummonedQuantityLimitStrategy`、`SummonedStatsStrategy`、`SummonedCostCtrlStrategy`、`SummonedUniqueTag`。

### 10.2 检测与引用

`SelfTagRequirements`、`TeammateTagRequirements`、`EnemyTagRequirements`、`bRequirementsIncludeCapturedSourceActorTags`、`ScopeQuote`、`ScopeDeathQuote`、`AbilityQuote`、`EffectAppliedRule`、`AssetTags`。

### 10.3 网络/预测/表现

`bAlwaysRelevant`、`bNetUseOwnerRelevancy`、`bNotRepActiveGameplayEffects`、`bNotRepPredictionKeyMap`、`bSupportRollback`、`PredictGameplayCues`、`bCanBePerceivedByAI`、`VisibleType`、`StencilConfig`、`DitheringConfig`、`bEnableDitheringTransparent`、`GameplayCues`、`bWalkable`、`bEnableSlide`、`bEndAgentTaskWhenDeath`、`bNeedKillTips`。

## 11. Equipment、Fly、CueOpt

### EquipmentTable

`EquipID`、`EquipHeroes`、`EquipSummoners`、`WeaponClasses`、`AmmoSpec`、`WeaponOwnedTags`、`bInitiallyEquip`、`bAllowMultiWielding`、`bIsWeakWielding`、`bEnableAutoReloadOnUnEquip`、`ReloadDelayTimeOnUnEquip`、`AttachSocket`、`SecondaryAttachSocket`、左右手 IK 位置/权重、`IKHandlingMethod`、`bCreateWeaponUI`、`bEnableBlockingNotice`、`MultiSpreadLevels`、`bReticleMultiSpreadLevel`、`EquipActionName`、`PresentGroup`、`GameplayCues`、`Delivery`。

### FlyConfigTable

`FlyModeID`、`AeroState`、`AeroConfig`、`Delivery`。

### CueOptTable

`CueTag`、`bEnableOptimization`、`CuePropertySet`、`bCustomOptimizationContext`、`Context`、`Delivery`。

## 12. Hero 表

### MarvelHeroTable

`HeroId`、`HeroTag`、`HeroSpecTags`、`PrimaryHeroId`、`TName`、`EnName`、`SculptNames`、`HeroDesc`、`bUsable`、`SortingParam`、`Gender`、`Role`、`bMultiRole`、`MultiRoleList`、`HeroType`、`MoveAbilityID`、`JumpAbilityID`、`CharacterClass`、`BodySize`、`ThreatValue`、`AttackMethod`、`Delivery`。

### MarvelHeroBaseAttributeTable

每行包含 `HeroId`、`TName`、`EnName`、`AttrSpecs`、`ExtendedConfig`、`AmmoIDs`、`Delivery`。每个 AttrSpec 包含形态 `TypeTag`、TurnSpeed、MoveSpeed、MoveBackSpeedRatio、Health/Shield/Armor 及 MoreAttributes/MoreEnergies。

## 13. 代表性证据资产

| 主题 | 文件 |
|---|---|
| 传送门 InlineConfig | `AbilitySystem/1018/101861/101861.json` |
| 传送门 Ability 关系 | `AbilitySystem/1018/101861/Ability_101861_BP.json` |
| Timeline/Montage | `AbilitySystem/1018/101861/Timeline_101861_Start.json` |
| 惩罚者 AbilityTable | `AbilitySystem/1014/1014_Table/1014_AbilityTable.json` |
| Projectile schema | `AbilitySystem/1014/1014_Table/1014_ProjectileAgentTable.json` |
| Equipment schema | `AbilitySystem/1014/1014_Table/1014_EquipmentTable.json` |
| Scope schema | `AbilitySystem/1014/1014_Table/1014_ScopeAgentTable.json` |
| Summon schema | `AbilitySystem/1014/1014_Table/1014_SummonerAgentTable.json` |
| 控制 Effect | `AbilitySystem/Common/GameEffect/Effect_1109.json` |
| 移速 Effect | `AbilitySystem/Common/GameEffect/Effect_1116.json` |
| 模式 BuffSpec | `AbilitySystem/GameMode/2208/Common/Tables/Common_2208_BuffSpecTable.json` |
| 英雄注册 | `Data/DataTable/MarvelHeroTable.json` |
| 英雄属性 | `Data/DataTable/MarvelHeroBaseAttributeTable.json` |
| 英雄技能 UI 入口 | `Data/DataTable/UI/HeroAbilityUITable.json` |
| 输入上下文 | `Data/Input/AbilityInputData/InputContext` |
| 模式表合并 | `AbilitySystem/GameMode/2208/BattleRules/2208_TableMerge.json` |

## 14. 数据审计查询建议

可以使用以下只读查询快速定位资产：

```powershell
# 查 AbilityID 的所有引用
rg "101861" "D:\FModel\Output\Exports\Marvel\Content\Marvel"

# 查某 Effect 被哪些 BuffSpec 引用
rg '"EffectID": 1116' "D:\FModel\Output\Exports\Marvel\Content\Marvel\AbilitySystem"

# 查状态 Tag 的生产者与消费者
rg 'Ability.Event.Block_Input' "D:\FModel\Output\Exports\Marvel\Content\Marvel\AbilitySystem"

# 查模式覆盖表
rg --files "D:\FModel\Output\Exports\Marvel\Content\Marvel\AbilitySystem\GameMode" | rg 'Table|Merge|Redirect'
```

