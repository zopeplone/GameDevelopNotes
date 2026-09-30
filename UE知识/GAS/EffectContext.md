https://www.thegames.dev/blog/creating-your-own-gameplay-effect-context/

下面以一个枚举和一个 `FName` 为例：

```cpp
// MyGameplayEffectContext.h
#pragma once

#include "GameplayEffectTypes.h"
#include "MyGameplayEffectContext.generated.h"

UENUM(BlueprintType)
enum class EMyHitType : uint8
{
    None,
    Physical,
    Magical
};

USTRUCT(BlueprintType)
struct FMyGameplayEffectContext : public FGameplayEffectContext
{
    GENERATED_BODY()

    UPROPERTY()
    EMyHitType HitType = EMyHitType::None;

    UPROPERTY()
    FName AttackName = NAME_None;

    virtual UScriptStruct* GetScriptStruct() const override
    {
        return StaticStruct();
    }

    virtual FMyGameplayEffectContext* Duplicate() const override
    {
        auto* NewContext = new FMyGameplayEffectContext(*this);
        if (HitResult.IsValid())
        {
            NewContext->HitResult = MakeShared<FHitResult>(*HitResult);
        }
        return NewContext;
    }

    virtual bool NetSerialize(FArchive& Ar, UPackageMap* Map, bool& bOutSuccess) override;
};

template<>
struct TStructOpsTypeTraits<FMyGameplayEffectContext>
    : TStructOpsTypeTraitsBase2<FMyGameplayEffectContext>
{
    enum
    {
        WithNetSerializer = true,
        WithCopy = true
    };
};
```

```cpp
// MyGameplayEffectContext.cpp
#include "MyGameplayEffectContext.h"
#include "UObject/CoreNet.h"

bool FMyGameplayEffectContext::NetSerialize(
    FArchive& Ar, UPackageMap* Map, bool& bOutSuccess)
{
    bool bBaseSuccess = false;
    const bool bBaseResult =
        FGameplayEffectContext::NetSerialize(Ar, Map, bBaseSuccess);

    uint8 HitTypeValue = static_cast<uint8>(HitType);
    Ar << HitTypeValue;

    if (Ar.IsLoading())
    {
        if (HitTypeValue > static_cast<uint8>(EMyHitType::Magical))
        {
            Ar.SetError();
        }
        else
        {
            HitType = static_cast<EMyHitType>(HitTypeValue);
        }
    }

    UPackageMap::StaticSerializeName(Ar, AttackName);

    bOutSuccess = bBaseSuccess && !Ar.IsError();
    return bBaseResult && bOutSuccess;
}
```

这里 `Duplicate()` 的关键是创建**派生类型**，复制全部字段，然后单独深拷贝 `TSharedPtr<FHitResult>`。文章使用 `AddHitResult(..., true)` 深拷贝；但 UE 5.6 中该函数会重置并可能重新设置 Origin。如果你单独设置过 Origin，它可能因此改变，所以直接复制 `HitResult` 更稳妥。见 [GameplayEffectTypes.h (line 394)](E:/UEProject/UE_5.6/Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/GameplayEffectTypes.h:394) 和 [GameplayEffectTypes.cpp (line 224)](E:/UEProject/UE_5.6/Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/GameplayEffectTypes.cpp:224)。

`FName` 使用 `UPackageMap::StaticSerializeName` 是为了明确采用网络名称序列化规则。网络 bit archive 中 `Ar << FName` 通常也能工作，但普通 `FArchive` 对 `FName` 的默认 `<<` 实现不写任何内容。最后，还需继承 `UAbilitySystemGlobals` 并覆写 `AllocGameplayEffectContext()` 返回 `new FMyGameplayEffectContext()`；UE 5.6 可在 Project Settings 的 Gameplay Abilities Settings 中指定该 Globals 类。见 [AbilitySystemGlobals.cpp (line 227)](E:/UEProject/UE_5.6/Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/AbilitySystemGlobals.cpp:227)。



Q: 为什么这里不加RepFlag了

A: 
因为我给的例子里，`HitType` 和 `AttackName` **每次都发送**。收发双方按固定顺序读取：先枚举，再 `FName`，所以不需要 `RepFlag` 来判断某个字段是否存在。基类自己的 `RepBits` 只管理基类字段，不能拿来表示派生类字段。见 [GameplayEffectTypes.cpp (line 237)](E:/UEProject/UE_5.6/Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/GameplayEffectTypes.cpp:237)。

如果大部分时候 `AttackName == NAME_None`，加一个派生类自己的 presence bit 可以省流量：

```
uint8 bHasAttackName = Ar.IsSaving() && !AttackName.IsNone();
Ar.SerializeBits(&bHasAttackName, 1);

if (bHasAttackName)
{
    UPackageMap::StaticSerializeName(Ar, AttackName);
}
else if (Ar.IsLoading())
{
    AttackName = NAME_None;
}
```

**何时加 `RepFlag` 是带宽选择，不是正确性要求。**字段少、总有值时固定发送更简单；字段多且经常为默认值时，用标志位按需发送更合适。按需发送时务必在读取端将缺席字段恢复为默认值，否则复用的 Context 可能保留旧值。

Q: Ar.IsLoading() 这段是做什么的

A :
这段代码做两件事：**把网络读到的字节写回 `HitType`，并检查它是不是合法枚举值**。`Ar << HitTypeValue` 在接收端只会修改局部变量 `HitTypeValue`，不会自动修改成员 `HitType`。`Ar.IsLoading()` 保证下面的赋值只在接收时执行。([dev.epicgames.com](https://dev.epicgames.com/documentation/unreal-engine/API/Runtime/Core/Serialization/FArchive?application_version=5.5&utm_source=openai))

以 `None = 0`、`Physical = 1`、`Magical = 2` 为例，收到 `1` 就转换为 `Physical`；收到 `200` 则调用 `Ar.SetError()`，将这次序列化标记为出错。([dev.epicgames.com](https://dev.epicgames.com/documentation/unreal-engine/API/Runtime/Core/Serialization/FArchive?application_version=5.5&utm_source=openai))

**如果整段删掉**，接收端虽然读到了字节，`HitType` 却仍是原值或默认值。**如果只删掉范围检查、保留赋值**，正常数据仍能工作，但异常值也会被转换成 `EMyHitType`，后续 `switch` 等逻辑可能遇到未处理的值。

这里的 `> Magical` 检查只适合示例中连续的 `0、1、2`。如果你的枚举有跳号，应该逐个判断允许的值，而不能只比较最大值。

Q:  NetSerialize是双向调用吗？接收和发送都调用？

A: 

对。`NetSerialize` 是**同一套代码用于发送和接收**，但一次调用只处理其中一个方向：

- 发送端调用时，`Ar.IsSaving()` 为真，`Ar << HitTypeValue` 把值写入网络数据。
- 接收端调用时，`Ar.IsLoading()` 为真，同一句 `Ar << HitTypeValue` 从网络数据读出值。随后才需要 `HitType = static_cast<EMyHitType>(HitTypeValue)`，把局部变量写回成员。([dev.epicgames.com](https://dev.epicgames.com/documentation/unreal-engine/API/Runtime/Core/FArchive?utm_source=openai))

例如服务器把 EffectContext 发给客户端时，服务器走写入路径，客户端走读取路径；并不是一次 `NetSerialize` 调用同时完成收和发。你在 [UE 5.6 的基类实现 (line 237)](E:/UEProject/UE_5.6/Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/GameplayEffectTypes.cpp:237) 中看到的 `if (Ar.IsSaving())` 也是这个用途：发送时决定哪些字段要写入，接收时则读取这些标志位，再按相同顺序读取字段。