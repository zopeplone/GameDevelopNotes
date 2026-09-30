**服务端执行 `PlayMontageAndWait` 后，通常会同步给其他客户端的模拟代理播放；但不会保证拥有该角色的客户端也通过这条路径播放。**

另外，`PlayMontageAndWait` 节点本身**不是 Multicast RPC**，它不会把 AbilityTask 对象复制到客户端。

**1. 节点本身只调用 ASC**

UE5.8 源码：

[AbilityTask_PlayMontageAndWait.h (line 64)](E:/UEProject/UE_5.8/Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/Abilities/Tasks/AbilityTask_PlayMontageAndWait.h:64)

核心实现位于 `Activate()`：

```
void UAbilityTask_PlayMontageAndWait::Activate()
{
    ...

    if (ASC->PlayMontage(
        Ability,
        Ability->GetCurrentActivationInfo(),
        MontageToPlay,
        Rate,
        StartSection,
        StartTimeSeconds) > 0.f)
    {
        ...
    }
}
```

UE5.8.2 对应源码：

[AbilityTask_PlayMontageAndWait.cpp (line 123)](C:/Users/lujingyu/AppData/Local/Temp/UE-src-check/Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/Abilities/Tasks/AbilityTask_PlayMontageAndWait.cpp:123)

所以调用链是：

```
PlayMontageAndWait
    -> UAbilityTask_PlayMontageAndWait::Activate()
    -> UAbilitySystemComponent::PlayMontage()
```

**2. ASC 的 `PlayMontage` 会写入复制数据**

[AbilitySystemComponent.h (line 1422)](E:/UEProject/UE_5.8/Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/AbilitySystemComponent.h:1422)：

```
/** Plays a montage and handles replication and prediction based on passed in ability/activation info */
virtual float PlayMontage(...);
```

内部首先在本端播放：

```
Duration = AnimInstance->Montage_Play(
    NewAnimMontage,
    InPlayRate,
    EMontagePlayReturnType::MontageLength,
    StartTimeSeconds);
```

然后服务端写入 `RepAnimMontageInfo`：

```
if (ShouldRecordMontageReplication())
{
    FGameplayAbilityRepAnimMontage& MutableRepAnimMontageInfo =
        GetRepAnimMontageInfo_Mutable();

    MutableRepAnimMontageInfo.Animation = Animation;
    MutableRepAnimMontageInfo.PlayInstanceId++;
    MutableRepAnimMontageInfo.PlayRate = ...;
    MutableRepAnimMontageInfo.Position = ...;
}
```

对应源码：

[AbilitySystemComponent_Abilities.cpp (line 3035)](C:/Users/lujingyu/AppData/Local/Temp/UE-src-check/Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/AbilitySystemComponent_Abilities.cpp:3035)

服务端还会强制 Avatar 进行网络更新：

```
if (IsOwnerActorAuthoritative())
{
    AbilityActorInfo->AvatarActor->ForceNetUpdate();
}
```

**3. `RepAnimMontageInfo` 是复制属性**

[AbilitySystemComponent.h (line 1775)](E:/UEProject/UE_5.8/Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/AbilitySystemComponent.h:1775)：

```
UPROPERTY(ReplicatedUsing=OnRep_ReplicatedAnimMontage)
FGameplayAbilityRepAnimMontage RepAnimMontageInfo;
```

并且注册为网络复制属性：

```
DOREPLIFETIME_WITH_PARAMS_FAST(
    UAbilitySystemComponent,
    RepAnimMontageInfo,
    Params);
```

UE5.8.2 源码：

[AbilitySystemComponent.cpp (line 1855)](C:/Users/lujingyu/AppData/Local/Temp/UE-src-check/Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/AbilitySystemComponent.cpp:1855)

复制内容包括：

```
Animation
PlayRate
Position
NextSectionID
PlayInstanceId
IsStopped
PredictionKey
SectionIdToPlay
```

定义见：

[GameplayAbilityRepAnimMontage.h (line 23)](E:/UEProject/UE_5.8/Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/Abilities/GameplayAbilityRepAnimMontage.h:23)

**4. 模拟客户端收到复制后会真正播放**

客户端执行：

```
void UAbilitySystemComponent::OnRep_ReplicatedAnimMontage()
{
    ...

    if (!AbilityActorInfo->IsLocallyControlled())
    {
        ...

        if (UAnimMontage* MontageToPlay =
            Cast<UAnimMontage>(ConstRepAnimMontageInfo.Animation))
        {
            PlayMontageSimulated(
                MontageToPlay,
                ConstRepAnimMontageInfo.PlayRate);
        }
    }
}
```

对应源码：

[AbilitySystemComponent_Abilities.cpp (line 3279)](C:/Users/lujingyu/AppData/Local/Temp/UE-src-check/Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/AbilitySystemComponent_Abilities.cpp:3279)

`PlayMontageSimulated()` 会调用：

```
AnimInstance->Montage_Play(NewAnimMontage, InPlayRate);
```

之后还会同步：

```
AnimInstance->Montage_SetPlayRate(...);
AnimInstance->Montage_SetPosition(...);
CurrentMontageStop(...);
```

**最终行为**

```
服务端执行 PlayMontageAndWait
    ├─ 服务端：播放蒙太奇
    ├─ 其他客户端的模拟代理：会通过 RepAnimMontageInfo 播放
    └─ 拥有该角色的客户端：不会在 OnRep 中执行模拟播放
```

原因是：

```
if (!AbilityActorInfo->IsLocallyControlled())
```

拥有者客户端 `IsLocallyControlled() == true`，因此跳过了 `PlayMontageSimulated()`。

所以实际结论是：

- **服务端调用后，其他客户端通常能看到该蒙太奇。**
- **拥有者客户端不能只依赖 `PlayMontageAndWait` 的复制来保证播放。**
- 拥有者客户端通常需要使用 `LocalPredicted`，让客户端也执行 Ability 和 `PlayMontageAndWait`；或者自己增加 Client RPC/Multicast 播放逻辑。
- 角色、ASC、AvatarActor 必须正常复制，客户端也必须拥有对应蒙太奇资源。