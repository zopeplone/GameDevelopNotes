[[UObject]] 是构成UE4对象系统的基础，PlayerController，Component，Actor等组件都继承自[[UObject]]，在C++中，需要使用指针引用这些对象。对于指针，有个常见的需求，就是当我们持有一个基类指针，我们想知道它是否可以转型成一个子类，此时可以用Cast。

一个例子如下，AController是基类，APlayerController是它的子类。Cast<APlayerController>会尝试把Exiting转型成APlayerController，如果其确实为APlayerController，则转型成功，得到APlayerController*类型的指针，如果它是AAIController等其他Controller，则转型失败，PC为空指针。

```cpp
void AGameMode::Logout( AController* Exiting )
{
	APlayerController* PC = Cast<APlayerController>(Exiting);
	if ( PC != nullptr )
	{
		RemovePlayerControllerFromPlayerCount(PC);
		AddInactivePlayer(PC->PlayerState, PC);
	}

	Super::Logout(Exiting);
}
```

这就是Cast的功能，用法简单且简洁。

除此之外，UE4还支持Interface的Cast。

## 纯C++要如何实现

先不考虑UE4的对象系统，尝试单纯用C++语法实现Cast功能，自然想到可以使用 **RTTI** (Run Time Type Identifiation)机制，即dynamic_cast。RTTI设计理念是根据class的vtbl实现，一种方式为把vtbl的第一个元素指向class的typeinfo信息，这也要求class有虚函数，这样才有vtbl。另外对每个class需额外存储typeinfo信息。

## UE4实现

UE的Cast大量使用了模板

查看Cast定义，发现Cast本质上是一个模板函数

```cpp
// Dynamically cast an object type-safely.
template <typename To, typename From>
FORCEINLINE To* Cast(From* Src)
{
	return TCastImpl<From, To>::DoCast(Src);
}
```

有两个模板参数，我们只需要提供To，From可以由函数参数自动推导得到。

继续跳转，TCastImpl也是模板函数，除了From和To，还有ECastType模板参数，其一般定义如下

```cpp
template <typename From, typename To>
struct TCastImpl<From, To, ECastType::InterfaceToInterface>
{
	FORCEINLINE static To* DoCast( From* Src )
	{
		return Src ? (To*)Src->_getUObject()->GetInterfaceAddress(To::UClassType::StaticClass()) : nullptr;
	}

	FORCEINLINE static To* DoCastCheckedWithoutTypeCheck( From* Src )
	{
		return Src ? (To*)Src->_getUObject()->GetInterfaceAddress(To::UClassType::StaticClass()) : nullptr;
	}
};
```

针对不同的CastType还有几个特化版本，下文介绍。

## CastType

ECastType有如下几种

```cpp
enum class ECastType
{
	UObjectToUObject,
	InterfaceToUObject,
	UObjectToInterface,
	InterfaceToInterface,
	FromCastFlags
};
```

可见主要分为[[UObject]]，Interface, FromCastFlags三类，前两者比较好理解，FromCastFlags表示使用预置的CastFlags比较是否可转型，只要位操作即可，速度最快。

我们首先要确定使用哪种CastType，然后才能决议使用哪个版本的特化TCastImpl。

ECastType使用TGetCastType模板函数得到，它的定义和几个特化版本如下

```cpp
template <typename From, typename To, bool bFromInterface = TIsIInterface<From>::Value, bool bToInterface = TIsIInterface<To>::Value, EClassCastFlags CastClass = TCastFlags<To>::Value>
struct TGetCastType
{
#if USTRUCT_FAST_ISCHILDOF_IMPL == USTRUCT_ISCHILDOF_STRUCTARRAY
	static const ECastType Value = ECastType::UObjectToUObject;
#else
	static const ECastType Value = ECastType::FromCastFlags;
#endif
};

template <typename From, typename To                           > struct TGetCastType<From, To, false, false, CASTCLASS_None> { static const ECastType Value = ECastType::UObjectToUObject;     };
template <typename From, typename To                           > struct TGetCastType<From, To, false, true , CASTCLASS_None> { static const ECastType Value = ECastType::UObjectToInterface;   };
template <typename From, typename To, EClassCastFlags CastClass> struct TGetCastType<From, To, true,  false, CastClass     > { static const ECastType Value = ECastType::InterfaceToUObject;   };
template <typename From, typename To, EClassCastFlags CastClass> struct TGetCastType<From, To, true,  true , CastClass     > { static const ECastType Value = ECastType::InterfaceToInterface; };
```

有两个问题要解决，一个是判断指针是否为IInterface，另一个是获取EClassCastFlags，这样就能使用下面的特化模板确定对应的ECastType。

## TIsInterface

```cpp
/**
 * Metafunction which detects whether or not a class is an IInterface.  Rules:
 *
 * 1. A UObject is not an IInterface.
 * 2. A type without a UClassType typedef member is not an IInterface.
 * 3. A type whose UClassType::StaticClassFlags does not have CLASS_Interface set is not an IInterface.
 *
 * Otherwise, assume it's an IInterface.
 */
template <typename T, bool bIsAUObject_IMPL = TPointerIsConvertibleFromTo<T, const volatile UObject>::Value>
struct TIsIInterface
{
	enum { Value = false };
};

template <typename T>
struct TIsIInterface<T, false>
{
	template <typename U> static char (&Resolve(typename U::UClassType*))[(U::UClassType::StaticClassFlags & CLASS_Interface) ? 2 : 1];
	template <typename U> static char (&Resolve(...))[1];

	enum { Value = sizeof(Resolve<T>(0)) - 1 };
};
```

判断指针是否为Interface有3个条件，首先[[UObject]]不是Interface，然后有Uclasstype且UClassType::StaticClassFlags有CLASS_Interface标记的才是IInterface。

TPointerIsConvertibleFromTo使用了C++原生的子类基类转型，这里用于判断指针是否为UObject*类型，其内部通过创建多个重载函数，用参数来判断给定数据类型是哪个。

## EClassCastFlags

```cpp
/**
 * Flags used for quickly casting classes of certain types; all class cast flags are inherited
 */
enum EClassCastFlags : uint64
{
	CASTCLASS_None = 0x0000000000000000,

	CASTCLASS_UField						= 0x0000000000000001,
	CASTCLASS_FInt8Property					= 0x0000000000000002,
	CASTCLASS_UEnum							= 0x0000000000000004,
	CASTCLASS_UStruct						= 0x0000000000000008,
	CASTCLASS_UScriptStruct					= 0x0000000000000010,
	CASTCLASS_UClass						= 0x0000000000000020,
	CASTCLASS_FByteProperty					= 0x0000000000000040,
	CASTCLASS_FIntProperty					= 0x0000000000000080,
	CASTCLASS_FFloatProperty				= 0x0000000000000100,
	CASTCLASS_FUInt64Property				= 0x0000000000000200,
	CASTCLASS_FClassProperty				= 0x0000000000000400,
```

UE4对很多内置类型分配了ClassCastFlags，比如UClass, APawn等等，都是一些位标记，存储在对应类UClass的ClassCastFlags属性中，而且Flags会被继承。TCastFlags模板函数可根据To类型获取到对应的Flag，原理也不复杂，UE4对每个类型都创建了特化的TCastFlags函数版本，如果To类型没有特化过，就返回CASTCLASS_None，表示该类型没有ClassCastFlags。

```cpp
#define DECLARE_ALL_CAST_FLAGS \
DECLARE_CAST_BY_FLAG(UField)							\
DECLARE_CAST_BY_FLAG(UEnum)								\
DECLARE_CAST_BY_FLAG(UStruct)							\
DECLARE_CAST_BY_FLAG(UScriptStruct)						\
DECLARE_CAST_BY_FLAG(UClass)							\
DECLARE_CAST_BY_FLAG(FProperty)							\
```

解决这些问题后，就可以正确选择TGetCastType特化模板，确定CastType了。下面分别介绍各个CastType的处理。

## FromCastFlags

```cpp
	// This is the cast flags implementation
	FORCEINLINE static To* DoCast( [[UObject]]* Src )
	{
		return Src && Src->GetClass()->HasAnyCastFlag(TCastFlags<To>::Value) ? (To*)Src : nullptr;
	}
```

对于一些引擎内置类型，可以查看From UClass的ClassCastFlags属性，是否有To的Flag。

## UObjectToUObject

```cpp
	FORCEINLINE static To* DoCast( [[UObject]]* Src )
	{
		return Src && Src->IsA<To>() ? (To*)Src : nullptr;
	}
```

UObject间的转型是最常见的情况，内部会使用IsA接口。该接口利用UE4对象系统，找到UObject对应的UClass，然后比较Frome和To的UClass是否有继承关系。最终进入到如下代码：

```cpp
#if USTRUCT_FAST_ISCHILDOF_COMPARE_WITH_OUTERWALK || USTRUCT_FAST_ISCHILDOF_IMPL == USTRUCT_ISCHILDOF_OUTERWALK
bool UStruct::IsChildOf( const UStruct* SomeBase ) const
{
	if (SomeBase == nullptr)
	{
		return false;
	}

	bool bOldResult = false;
	for ( const UStruct* TempStruct=this; TempStruct; TempStruct=TempStruct->GetSuperStruct() )
	{
		if ( TempStruct == SomeBase )
		{
			bOldResult = true;
			break;
		}
	}
```

使用For循环，不断找From的SuperClass，直到发现和To的UClass相等。

## UObjectToInterface

UE4的UObject和IInterface也可以相互转型，关于Interface细节可见，[接口 | 虚幻引擎文档 (unrealengine.com)](https://link.zhihu.com/?target=https%3A//docs.unrealengine.com/4.27/zh-CN/ProgrammingAndScripting/GameplayArchitecture/Interfaces/)，Cast实现代码如下：

```cpp
	FORCEINLINE static To* DoCast( UObject* Src )
	{
		return Src ? (To*)Src->GetInterfaceAddress(To::UClassType::StaticClass()) : nullptr;
	}
```

To::UClassType::StaticClass()是IInterface对应的UInterface的UClass，可能有点绕。

UClass的Interfaces属性记录了该Class实现的所有Interface，并不包含Interface的父类。因此通过Interfaces可建立UObject到IInterface的联系。

GetInterfaceAddress函数实现如下

```cpp
void* UObjectBaseUtility::GetInterfaceAddress( UClass* InterfaceClass )
{
	void* Result = NULL;

	if ( InterfaceClass != NULL && InterfaceClass->HasAnyClassFlags(CLASS_Interface) && InterfaceClass != UInterface::StaticClass() )
	{
		// Script interface
		if ( !InterfaceClass->HasAnyClassFlags(CLASS_Native) )
		{
			if ( GetClass()->ImplementsInterface(InterfaceClass) )
			{
				// if it isn't a native interface, the address won't be different
				Result = this;
			}
		}
		// Native interface
		else
		{
			for( UClass* CurrentClass=GetClass(); Result == NULL && CurrentClass != NULL; CurrentClass = CurrentClass->GetSuperClass() )
			{
				for (TArray<FImplementedInterface>::TIterator It(CurrentClass->Interfaces); It; ++It)
				{
					// See if this is the implementation we are looking for, and it was done natively, not in K2
					FImplementedInterface& ImplInterface = *It;
					if ( !ImplInterface.bImplementedByK2 && ImplInterface.Class->IsChildOf(InterfaceClass) )
					{
						Result = (uint8*)this + It->PointerOffset;
						break;
					}
				}
			}
		}
	}

	return Result;
}
```

分蓝图Interface和C++ Interface两部分，主要看433行的C++代码，蓝图最终处理流程也类似。需要遍历UObject的所有SuperClass，然后遍历实现的Interface，对于每个Interface，还要遍历Interface的SuperClass，看是否和To Interface的UClass相等，需要遍历多次，是个复杂度比较高的操作。

另外注意Interface实际上通过C++多继承实现，因此IInterface的指针地址和UObject并不相同，所以要对UObject地址加上Offset，作为IInterface地址。

## InterfaceToUObject

```cpp
template <typename From, typename To>
struct TCastImpl<From, To, ECastType::InterfaceToUObject>
{
	FORCEINLINE static To* DoCast( From* Src )
	{
		To* Result = nullptr;
		if (Src)
		{
			UObject* Obj = Src->_getUObject();
			if (Obj->IsA<To>())
			{
				Result = (To*)Obj;
			}
		}
		return Result;
	}
```

通过IInterface的_getUObject()方法可获取到对应的UObject，接下来只要用UObjectToUObject的Cast方式即可。

## InterfaceToInterface

```cpp
	FORCEINLINE static To* DoCast( From* Src )
	{
		return Src ? (To*)Src->_getUObject()->GetInterfaceAddress(To::UClassType::StaticClass()) : nullptr;
	}
```

同样的，先把IInterface转成UObject，再通过UObjectToInterface的Cast即可。

  

以上为UE4 Cast的基本实现原理，适用于指针Cast，除此之外，UE4也支持TWeakObjectPtr，TSubclassOf的Cast，最终都是用的指针Cast。

## Cast的开销

1. 需要不断查找UClass的父类，时间复杂度可认为是O(N)，N是class的继承链长度
2. 模板开销，对每个<From, To>组合都要实例化一个模板，增大代码段
3. UClass要记录SuperStruct，其实不能算Cast导致的开销，UE对象系统本来就这么设计的