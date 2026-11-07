# MarvelAbilityItem 完整配置类型

## 1. 数据范围与结论

本文扫描了 `AbilitySystem` 目录下全部导出的 JSON：

- 包含 `MarvelAbilityItem` 的文件：1163 个
- 成功解析并参与字段归并的 `MarvelAbilityItem`：1145 个
- `MarvelAbilityItem.Properties` 顶层字段：82 个

下面的类型是根据 FModel JSON、对象引用、枚举文本和字段实际值反推的 UE 类型。原游戏 C++ 头文件未包含在解包资源中，因此 `FMarvel...` 结构体名称是语义化命名，不保证与网易源码中的真实名称完全相同；字段名称、嵌套关系和数组元素结构来自实际导出数据。

`ExtendConfigInline` 是多态 UObject，不属于固定的 AbilityItem 字段展开范围。它可能是 `Config_109`、`Config_105471`、`PyConfig_102971` 等任意技能专属配置类。

## 2. 基础辅助类型

```cpp
// FModel 对 UObject/Class/Asset 引用的统一导出形式。
struct FMarvelObjectReference
{
    FString ObjectName;
    FString ObjectPath;
};

struct FMarvelTag
{
    FGameplayTag TagName;
};

// JSON 可能导出为 CultureInvariantString，或者 StringTable 四元组。
struct FMarvelText
{
    TOptional<FString> CultureInvariantString;
    TOptional<FString> TableId;
    TOptional<FString> Key;
    TOptional<FString> SourceString;
    TOptional<FString> LocalizedString;
};
```

### AbilityActivateData

全量数据中 `ActivateData` 只引用了一个类：`AbilityActivateData_Base`。61 个实例全部以对应 `MarvelAbilityItem` 为 `Outer`，其可见类型为：

```cpp
enum class EAbilityActivatePriority
{
    BoundAbility,
    LevelInteractiveAbility,
    SelectAbility,
    CustomPriority
    // 可能还有当前资源未使用的枚举成员。
};

UCLASS(EditInlineNew, DefaultToInstanced) // 宏为基于资源形态的推断
class UAbilityActivateData_Base : public UObject
{
    GENERATED_BODY()

public:
    EAbilityActivatePriority ActivatePriorityType;
    EAbilityInputBinds InputBindConflict;
    int32 Priority;
};
```

显式配置统计：

| 字段 | 类型 | 显式出现数 | 已观察值 |
|---|---|---:|---|
| `ActivatePriorityType` | `EAbilityActivatePriority` | 47 | `BoundAbility`、`LevelInteractiveAbility`、`SelectAbility`、`CustomPriority` |
| `InputBindConflict` | `EAbilityInputBinds` | 4 | `LevelInteractive`、`AbilityShortCut8` |
| `Priority` | `int32` | 3 | `301`、`500`、`1000` |

其余实例没有导出某字段，表示使用类默认对象（CDO）的默认值，而不是字段不存在。

资源层面可以确认它是 AbilityItem 包内的内嵌 UObject 子对象：对象 `Outer` 是 `MarvelAbilityItem`，引用路径也指向同一个 AbilityItem 包。仅凭 cooked 资源不能证明源码属性究竟写了 `UPROPERTY(Instanced)`，还是配置类使用了 `DefaultToInstanced`，但运行时所有样本都表现为 Instanced/Inline 子对象。

## 3. 输入结构

```cpp
struct FMarvelAbilityInputConfig
{
    EAbilityInputBinds InputBind;

    // 仅部分 AbilityItem 导出，未写出的字段使用类默认值。
    EAbilityInputType SkillActivationMode;
    bool bTriggerOnRelease;
    float MaxHoldTime;
    float AutoTriggerTime;
};

struct FMarvelActionTipsParam
{
    EAbilityInputBinds InputBind;
    TObjectPtr<UInputAction> BindInputAction;
    FText Tips;
    FName ActionName;
    FText TipsInputStateA;
    FText TipsInputStateB;
    int32 InputState;
    int32 Priority;
    EHorizontalAlignment HorizontalAlignment;
    EAbilityInputBinds GamepadBind;
    bool bHighlight;
};

struct FMarvelBindInputConfig
{
    EBindInputType BindType;
    EAbilityInputType SkillActivationMode;
    FName MethodName;
    TObjectPtr<UInputAction> BindInputAction;
    bool bShouldUseSpecialAction;
    TObjectPtr<UInputAction> GamepadInputAction;
    bool bBlockOtherInput;
    float DelayTriggerTime;
    bool bUseIgnoreInputTemplateMask;
    int32 IgnoreInputTemplateMasks;
    TArray<FGameplayTag> TriggerBlockedTags;
    TArray<FGameplayTag> TriggerBlockedTags_Blocking;
    FMarvelActionTipsParam ActionTipsParam;
};

struct FMarvelLegacyBindInput
{
    FMarvelAbilityInputConfig InputConfig;
    FName MethodName;
    float DelayTriggerTime;
    bool bBlockAbilityInput;
    FMarvelActionTipsParam BindInputActionTipsParam;
};
```

说明：`BindInputConfigs` 是当前较完整的 Enhanced Input 绑定结构，`BindInputs` 更像兼容旧系统的输入绑定结构。两者都实际存在，不能合并成同一个字段。

## 4. 阶段、消耗与冷却结构

```cpp
struct FMarvelCostAttrAction
{
    EMarvelAbilityStage AbilityStage;
    FGameplayTag CustomTagEvent;

    // FModel 只导出了整数。该字段很可能是位标志枚举。
    int32 CostAction;
};

enum class EMarvelAbilityStage
{
    None,
    Active,
    Missile,
    Break,
    End,
    Custom,
    TagEvent
    // 实际枚举可能还有未被当前资源使用的成员。
};
```

`CostAction` 当前可见值包含 `1`、`8`、`32` 等，因此更像按位组合的动作标志，而不是普通顺序枚举。没有原游戏头文件时不应擅自给每个数字命名。

## 5. GameplayCue、连段与表现结构

```cpp
struct FMarvelAbilityGameplayCue
{
    FGameplayTag GameplayCueTag;
    EAbilityCueStage AddStage;
    EAbilityCueStage RemoveStage;
    bool bPredictablyRemovePredictedCue;
};

struct FMarvelTimelineMontageEntry
{
    TObjectPtr<UAnimTimeline> Timeline;
    int32 AnimResIndex;
};

struct FMarvelAbilityComboSection
{
    bool bRequireHit;
    bool bResetWhenInterrupted;
    bool bResetWhenInterruptedAfterMissile;
    int32 ComboIDWhenHit;
    int32 ComboIDWhenNotHit;
    int32 ComboIDWhenInterrupted;
    float ComboTimeout;
};

struct FMarvelAnimFrameFreezeConfig
{
    float EnemyAnimRateScale;
    float EnemyAnimRateDuration;
    float SceneAnimRateScale;
    float SceneAnimRateDuration;
};

struct FMarvelHighlightComboScope
{
    FString Key;       // JSON Map 的键，常见为 "0"、"1"。
    int32 Value;       // Scope ID。
};

struct FMarvelPPTransparentDistanceOffset
{
    float DistanceOffset;
};
```

## 6. MarvelAbilityItem 完整顶层类型

```cpp
UCLASS(BlueprintType)
class UMarvelAbilityItem : public UObject
{
    GENERATED_BODY()

public:
    // ------------------------------------------------------------------
    // 核心 Ability 与多态扩展配置
    // ------------------------------------------------------------------

    // 必有。实际值是 UGameplayAbility 派生蓝图类。
    TSubclassOf<UGameplayAbility> GrantedAbility;

    // 仅发现 1 个样本。值为 GameplayAbilityDataSet_Vector 类。
    TSubclassOf<UGameplayAbilityDataSet> GrantedAbilityDataSet;

    // 多态 Instanced UObject。可能是 Config_xxx、PyConfig_xxx 或
    // MarvelAbilityConfig；具体结构由对应技能代码定义。
    TObjectPtr<UObject> ExtendConfigInline;

    // 可选的激活数据对象，例如 AbilityActivateData_Base。
    TObjectPtr<UObject> ActivateData;

    // ------------------------------------------------------------------
    // 输入
    // ------------------------------------------------------------------

    FMarvelAbilityInputConfig ActivateInput;
    TArray<FMarvelBindInputConfig> BindInputConfigs;
    TArray<FMarvelLegacyBindInput> BindInputs;

    EAbilityInputBinds ConfirmInput;
    EAbilityInputBinds CancelInput;
    TArray<EAbilityInputBinds> StandByActivateInput;

    FMarvelActionTipsParam ConfirmTipsParam;
    FMarvelActionTipsParam CancelTipsParam;

    float CancelInputDelayTime;
    float CustomActivateInterval;
    bool bStopLoopActivateWhenUserCancel;

    TArray<TObjectPtr<UInputAction>> CancelLoopAbilityActions;
    TArray<EAbilityInputBinds> BlockInputBinds;
    TArray<EAbilityInputBinds> KeyBoardBlockInputBinds;
    int32 KeyConsumptionPriority;

    // ------------------------------------------------------------------
    // 技能阶段、Timeline 与连段
    // ------------------------------------------------------------------

    TArray<FMarvelTimelineMontageEntry> TimelineToPlayMontage;
    TArray<FMarvelAbilityComboSection> AbilityComboSections;

    EMarvelAbilityStage AutoBeginGather;
    TArray<float> GatherCallbackValues;

    bool bUsingDifferentComboRange;
    TArray<FMarvelHighlightComboScope> HighlightComboScopes;
    int32 ScopeIDNeedHighlight;

    // ------------------------------------------------------------------
    // 冷却提交时机
    // ------------------------------------------------------------------

    EMarvelAbilityStage ReadyCooldownStage;
    FGameplayTag ReadyCooldownStageTag;
    EMarvelAbilityStage CommitCooldownStage;
    FGameplayTag CommitCooldownStageTag;
    bool bCommitCooldownOnDeathInterrupt;

    // ------------------------------------------------------------------
    // Cost/资源
    // ------------------------------------------------------------------

    EMarvelAbilityStage ReadyCostStage;
    FGameplayTag ReadyCostStageTag;
    TArray<FMarvelCostAttrAction> CostAttrActions;

    FGameplayTag CostClipTypeTag;
    FGameplayTag CostRequireTag;
    FGameplayTag CostIgnoreTag;

    float InitValue;
    float MaxValue;
    float RegenValue;
    float ActivateCostValue;
    float CheckActivateCostValue;
    float MinimalActivateCostValue;
    float ReduceValue;
    float RecoverDelayTime;

    float DamageRegen;
    float TreatRegen;

    bool PrimaryCost;
    bool bInitActiveCost;
    bool bDefaultRegen;
    bool bRestartRegenWhernCost; // 原配置拼写就是 Whern。
    bool bCanActiveWithoutEnoughEnergy;
    bool bEndWithoutEnoughEnergy;
    bool bFollowIgnoreCost;
    bool bSlackCheckActivateCost;
    float SlackCheckCostTime;

    // ------------------------------------------------------------------
    // GameplayCue
    // ------------------------------------------------------------------

    TArray<FMarvelAbilityGameplayCue> AbilityGameplayCues;

    // ------------------------------------------------------------------
    // 镜头、震屏、顿帧
    // ------------------------------------------------------------------

    bool EnableCameraShake;
    bool AcivateAbilityShake; // 原配置拼写就是 Acivate。
    bool AutoActiveCameraInterp;
    // 虽然名字像 bool，实际保存触发 HUD 动画的 Ability Stage。
    EMarvelAbilityStage AutoShakeHUD;
    bool bChooseCameraShakeByCombo;

    TSubclassOf<UCameraShakeBase> AbilityCameraShakeClass;
    TSubclassOf<UCameraShakeBase> HitCameraShakeClass;
    TSubclassOf<UCameraShakeBase> HitEnemyCameraShakeClass;
    TArray<TSubclassOf<UCameraShakeBase>> HitEnemyCameraShakeClassesByCombo;

    TObjectPtr<UObject> CameraInterpAsset; // 实际资产类型名为 CameraInterp。
    EMarvelAbilityUIHudAnim ShakeHUDType;

    bool EnableAnimFrameFreeze;
    TArray<FMarvelAnimFrameFreezeConfig> AnimFrameFreezeConfig;
    FMarvelPPTransparentDistanceOffset PPTransparentDistanceOffset;

    // ------------------------------------------------------------------
    // 命中类型开关
    // ------------------------------------------------------------------

    bool HitCharactersAndNonShieldSummons;
    bool HitShieldSummons;
    bool HitScenes;
    bool HitDestructibleScenes;

    // ------------------------------------------------------------------
    // 失败提示、UI 高亮与复制
    // ------------------------------------------------------------------

    bool bDisableAudioOnFailCooldown;
    bool bDisableAudioOnFailCost;
    bool bDisableAudioOnFailBlock;
    bool bDisableAudioOnOtherCanActivate;
    bool bDisableAudioOnUserLogic;

    bool bOnlyHighlightWhenActivated;
    bool bReplicateDataSet;
};
```

### `AutoShakeHUD` 类型说明

实际 JSON 有两个字段：

```json
"AutoShakeHUD": "EMarvelAbilityStage::Missile",
"ShakeHUDType": "EMarvelAbilityUIHudAnim::Melee"
```

因此 `AutoShakeHUD` 是枚举，并非 `bool`：

```cpp
EMarvelAbilityStage AutoShakeHUD;
EMarvelAbilityUIHudAnim ShakeHUDType;
```

字段名虽然像布尔值，但原项目没有 `b` 前缀，实际保存的是触发 HUD Shake 的 Ability Stage。

## 7. 82 个顶层字段总表

`出现数`表示 1145 个成功解析对象中显式导出该字段的对象数。未导出通常表示使用 CDO 默认值，并不表示 C++ 类中没有这个属性。

| 字段 | 推断类型 | 出现数 |
|---|---:|---:|
| `GrantedAbility` | `TSubclassOf<UGameplayAbility>` | 1145 |
| `GrantedAbilityDataSet` | `TSubclassOf<UGameplayAbilityDataSet>` | 1 |
| `ExtendConfigInline` | `TObjectPtr<UObject>`，多态 Instanced 配置 | 1080 |
| `ActivateData` | `TObjectPtr<UObject>` | 61 |
| `ActivateInput` | `FMarvelAbilityInputConfig` | 1025 |
| `BindInputConfigs` | `TArray<FMarvelBindInputConfig>` | 873 |
| `BindInputs` | `TArray<FMarvelLegacyBindInput>` | 169 |
| `ConfirmInput` | `EAbilityInputBinds` | 42 |
| `CancelInput` | `EAbilityInputBinds` | 59 |
| `StandByActivateInput` | `TArray<EAbilityInputBinds>` | 29 |
| `ConfirmTipsParam` | `FMarvelActionTipsParam` | 35 |
| `CancelTipsParam` | `FMarvelActionTipsParam` | 42 |
| `CancelInputDelayTime` | `float` | 12 |
| `CustomActivateInterval` | `float` | 20 |
| `bStopLoopActivateWhenUserCancel` | `bool` | 2 |
| `CancelLoopAbilityActions` | `TArray<TObjectPtr<UInputAction>>` | 3 |
| `BlockInputBinds` | `TArray<EAbilityInputBinds>` | 4 |
| `KeyBoardBlockInputBinds` | `TArray<EAbilityInputBinds>` | 2 |
| `KeyConsumptionPriority` | `int32` | 2 |
| `TimelineToPlayMontage` | `TArray<FMarvelTimelineMontageEntry>` | 825 |
| `AbilityComboSections` | `TArray<FMarvelAbilityComboSection>` | 132 |
| `AutoBeginGather` | `EMarvelAbilityStage` | 13 |
| `GatherCallbackValues` | `TArray<float>` | 8 |
| `bUsingDifferentComboRange` | `bool` | 2 |
| `HighlightComboScopes` | `TMap<int32,int32>` 或导出键值数组 | 2 |
| `ScopeIDNeedHighlight` | `int32` | 36 |
| `ReadyCooldownStage` | `EMarvelAbilityStage` | 172 |
| `ReadyCooldownStageTag` | `FGameplayTag` | 3 |
| `CommitCooldownStage` | `EMarvelAbilityStage` | 376 |
| `CommitCooldownStageTag` | `FGameplayTag` | 1 |
| `bCommitCooldownOnDeathInterrupt` | `bool` | 1 |
| `ReadyCostStage` | `EMarvelAbilityStage` | 605 |
| `ReadyCostStageTag` | `FGameplayTag` | 3 |
| `CostAttrActions` | `TArray<FMarvelCostAttrAction>` | 753 |
| `CostClipTypeTag` | `FGameplayTag` | 227 |
| `CostRequireTag` | `FGameplayTag` | 11 |
| `CostIgnoreTag` | `FGameplayTag` | 3 |
| `InitValue` | `float` | 167 |
| `MaxValue` | `float` | 238 |
| `RegenValue` | `float` | 171 |
| `ActivateCostValue` | `float` | 181 |
| `CheckActivateCostValue` | `float` | 192 |
| `MinimalActivateCostValue` | `float` | 1 |
| `ReduceValue` | `float` | 21 |
| `RecoverDelayTime` | `float` | 17 |
| `DamageRegen` | `float` | 72 |
| `TreatRegen` | `float` | 47 |
| `PrimaryCost` | `bool` | 16 |
| `bInitActiveCost` | `bool` | 135 |
| `bDefaultRegen` | `bool` | 15 |
| `bRestartRegenWhernCost` | `bool` | 36 |
| `bCanActiveWithoutEnoughEnergy` | `bool` | 23 |
| `bEndWithoutEnoughEnergy` | `bool` | 62 |
| `bFollowIgnoreCost` | `bool` | 3 |
| `bSlackCheckActivateCost` | `bool` | 8 |
| `SlackCheckCostTime` | `float` | 2 |
| `AbilityGameplayCues` | `TArray<FMarvelAbilityGameplayCue>` | 326 |
| `EnableCameraShake` | `bool` | 291 |
| `AcivateAbilityShake` | `bool` | 49 |
| `AutoActiveCameraInterp` | `bool` | 63 |
| `AutoShakeHUD` | `EMarvelAbilityStage` | 185 |
| `ShakeHUDType` | `EMarvelAbilityUIHudAnim` | 187 |
| `bChooseCameraShakeByCombo` | `bool` | 17 |
| `AbilityCameraShakeClass` | `TSubclassOf<UCameraShakeBase>` | 48 |
| `HitCameraShakeClass` | `TSubclassOf<UCameraShakeBase>` | 164 |
| `HitEnemyCameraShakeClass` | `TSubclassOf<UCameraShakeBase>` | 216 |
| `HitEnemyCameraShakeClassesByCombo` | `TArray<TSubclassOf<UCameraShakeBase>>` | 15 |
| `CameraInterpAsset` | `TObjectPtr<UObject/CameraInterp>` | 131 |
| `EnableAnimFrameFreeze` | `bool` | 14 |
| `AnimFrameFreezeConfig` | `TArray<FMarvelAnimFrameFreezeConfig>` | 13 |
| `PPTransparentDistanceOffset` | `FMarvelPPTransparentDistanceOffset` | 1 |
| `HitCharactersAndNonShieldSummons` | `bool` | 277 |
| `HitShieldSummons` | `bool` | 225 |
| `HitScenes` | `bool` | 110 |
| `HitDestructibleScenes` | `bool` | 107 |
| `bDisableAudioOnFailCooldown` | `bool` | 25 |
| `bDisableAudioOnFailCost` | `bool` | 22 |
| `bDisableAudioOnFailBlock` | `bool` | 33 |
| `bDisableAudioOnOtherCanActivate` | `bool` | 3 |
| `bDisableAudioOnUserLogic` | `bool` | 12 |
| `bOnlyHighlightWhenActivated` | `bool` | 6 |
| `bReplicateDataSet` | `bool` | 1 |

## 8. 不属于 MarvelAbilityItem 固定类型的内容

下面这些对象经常和 AbilityItem 导出在同一个 JSON 文件里，但不能并入 `UMarvelAbilityItem`：

1. `Config_xxx`：C++ 技能专属扩展配置。
2. `PyConfig_xxx`：Python/脚本技能专属扩展配置。
3. `MarvelAbilityConfig`：通用扩展配置基类或默认配置。
4. `AbilityActivateData_Base` 及派生对象：激活阶段数据对象。
5. `Ability_xxx_BP_C`：真正执行技能行为的 GameplayAbility 类 CDO。
6. `AnimTimeline`：动画、事件、判定和 Buff 时序资产。

它们与 AbilityItem 的组合关系是：

```text
MarvelAbilityItem
├─ GrantedAbility       -> GameplayAbility 类
├─ ExtendConfigInline   -> Config_xxx / PyConfig_xxx
├─ ActivateData         -> AbilityActivateData 派生对象
├─ TimelineToPlayMontage-> AnimTimeline 数组
├─ 输入配置
├─ Cost/CD 阶段配置
└─ 镜头、震屏、顿帧等公共表现配置
```

## 9. 实现建议

复刻时不建议直接创建一个拥有全部英雄专属字段的巨型结构。应保持原项目类似的分层：

```cpp
UCLASS(Abstract, BlueprintType, EditInlineNew, DefaultToInstanced)
class UMarvelAbilityExtendConfig : public UObject
{
    GENERATED_BODY()
};

UCLASS(BlueprintType)
class UMarvelAbilityItem : public UPrimaryDataAsset
{
    GENERATED_BODY()

public:
    UPROPERTY(EditDefaultsOnly)
    TSubclassOf<UGameplayAbility> GrantedAbility;

    UPROPERTY(EditDefaultsOnly, Instanced)
    TObjectPtr<UMarvelAbilityExtendConfig> ExtendConfigInline;

    // 其余公共字段按上面的完整类型声明。
};
```

每种复杂技能再定义独立扩展类，例如 `UConfig_105471`。这样可以避免让所有技能携带大量无意义字段，同时允许数据资产直接内嵌编辑技能专属参数。
