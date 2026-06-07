
先来看一个ts脚本
```ts
const AssetPath = "/Game/BP_Character.BP_Character_C";
const Cls = UE.Class.Load(AssetPath);
const BP_CharacterClass = puerts.blueprint.tojs(Cls);
class TsCharacter implements UE.Game.BP_Character {
	jsNativeObj: {a:number,b:string};
	BeginPlay(){
		const LevelName = UE.GameplayStatics.GetCurrentLevelName(this.GetWorld());
		console.log(`LevelName: ${LevelName}`);
		const CMC = this.CharacterMovement;
		CMC.bPressedJump = true;
		const Name = this.GetName();
		console.log(Name);
		const Vec = new UE.Vector(1,2,3);
		this.jsNativeObj = {a:1,b:"bbbbb"};
		this.OnMovementModeChanged.Add((Character:UE.Character, PrevMovementMode:UE.EMovementMode) =>{
			console.log(`MovementMode From ${PrevMovementMode} To ${Character.MovementMode}`);
			const isSame = Character === this; // true
		})
	}
}

blueprint.mixin(TsCharacter, BP_CharacterClass);
```

在Puerts的官方文档下，有这样一段话

> **一个UE对象传入到ts**，ts侧会建立一个stub (ts)对象与之相对应（ts调用这个stub对象会被转发到真实的UE原生调用），而且在puerts中他们的生命周期间的关系有两种。
> stub对象由js gc管理，stub对象持有ue对象的强引用（下称“stub对象持有ue对象”）
	如果stub对象在ts无引用，将会被gc，进而释放对ue对象的强引用
	如果进一步在ue引擎也没有该ue对象，该ue对象会被gc
	ue对象由ue gc管理，ue对象持有stub对象的强引用（下称“ue对象持有stub对象”）
	如果ue对象在ue引擎无引用，该ue对象会被gc，进而释放对stub对象的强引用- 如果进一步在ts也没有引用该stub对象，该stub对象会被gc

在笔者刚接触puerts时，看完这段话，写出上面这段代码，对于这些UE和js的对象，很容易就会产生一些疑问，
1. `一个UE对象传入到ts` ,puerts怎么知道一个对象怎么传入到js的？
2. 在 `const CMC = this.CharacterMovement` puerts是怎么知道我拿了`CharacterMovement`,又怎么知道`CharacterMovement`是UE内的变量，从而在我`CMC.bPressedJump = true`时，在引擎内部的bPressedJump设置为true
3. puerts是如何区分jsNative变量和UE变量的？
4. `OnMovementModeChanged`这个`MultiCastDelegate`为什么可以直接传入箭头函数？
5. 为什么可以直接new UE.Vector()

这些问题汇总一下，就是js与UE的对象映射机制


# js能够访问到UE数据的路径

对于第一个问题，`puerts怎么知道一个对象怎么传入到js的`，
我们可以反过来从js层来看**主动侧**，js怎么访问UE？UE实际上是不知道js层的，**只有js主动去与UE交互**；我们知道puerts对js层是完全控制的，puerts只需要知道js中如何取UE对象即可，只要把所有js访问UE的路径都枚举一遍，在这些路径上设下**拦截**即可

我们还是分析一下上述脚本，找找从js获取到UE对象的**路径**有哪些
## 1. 从UE Object中获取暴露的反射类
`const Cls = UE.Class.Load(AssetPath);`， `UE.GameplayStatics.GetCurrentLevelName`
UE.Class(UClass)和UE.GameplayStatics(UGameplayStatics) 都是UE类型系统中的反射类，这个路径是用户js从"ue"模块中获取UE类型系统暴露的类型
在js中，获取UE这个`Object`的方式是
```ts
import * as UE from 'ue';
// 或者
const UE = require("ue");
```
`'ue'`这个模块是从哪里来的？在笔者深入源码寻找发现，在`uelazyload.js`中，给UE这个模块对象设置了一个拦截器，每次UE.xxx时，都会从C++侧去取"xxx" 这个UE类型，然后转成js中的class；比如我们获取`UE.GameplayStatics`，链路如下
1. UE对象proxy懒加载拦截
2. 调用FJsEnvImpl::LoadUEType
3. LoadUEType调用 UE的 `FindObject<T>(ANY_PACKAGE, InShortName);` InShortName就是 "`GameplayStatics`"，T是`UClass`
4. 找到GameplayStatics的UClass对象后，调用FJsEnvImpl::UETypeToJsClass，其中返回一个Js Function，是Js构造函数模拟的UE类（这个函数是怎么创建出这个JsFunction的请见详解篇）
返回的这个Js构造函数模拟的UE类是什么？我们看看这个UE类在js中的构造函数对象结构
如果不清楚js prototype的 可以看这个视频 [JavaScript 原型和原型链](https://www.bilibili.com/video/BV1rM4y147iy)

```cpp
ctor (v8::Function)                          ← new ctor(...) 来创建实例
│
├── 静态成员/函数 (直接在 ctor 上)
│   ├── ctor.StaticClass()                   ← 返回UClass*
│   ├── ctor.StaticStruct()                  ← (仅 UScriptStruct) 返回自身的 UScriptStruct* 包装
│   ├── ctor.Find(name, outer?)              ← 查找已有对象
│   ├── ctor.Load(path, unescape?)           ← 加载一个UAsset，转为自身类型的对象
│   ├── ctor.SomeStaticUFunc()               ← UFUNCTION static函数
│   └── ctor.SomeCppRegisteredStatic()       ← 通过cpp模板反射（静态反射）注册的static函数
│
├── ctor.prototype                           ← 所有实例共享的原型对象
│   ├── .SomeInstanceUFunc()                 ← UFUNCTION实例方法
│   ├── .SomeCppRegisteredMethod()           ← 通过cpp模板反射注册的成员方法
│   ├── .SomeExtensionMethod()               ← 通过cpp模板反射注册的成员扩展方法
│   ├── .SomeProp (getter/setter)            ← UE 成员变量
│   └── .SomeCppVar (getter/setter)          ← 通过cpp模板反射注册的成员变量
│
└── 内部槽位（通过 InternalFieldCount=4 预留）
    ├── InternalField[0]: UObject* 或 void* 指针（低32位）
    ├── InternalField[1]: UObject* 或 void* 指针（高32位/类型标记）
    ├── InternalField[2]: (预留)
    └── InternalField[3]: (预留)
```
有了这个js ue类，我们就可以解答如下两个问题
1. `UE.GameplayStatics.GetCurrentLevelName` 这个方法是从哪里来的
UGameplayStatics标识static UFUNCTION `GetCurrentLevelName` ，静态函数指针 被puerts直接设置在原型上
2. 怎么知道`CharacterMovement`是UE内的变量
puerts在加载ACharacter类到js时，把`CharacterMovement`标识为UE的成员变量，其中有getter和setter；
PS: 是在我们mixin的时候，puerts加载ACharacter类到js中

## 2. js中获取UE的对象
`const Cls = UE.Class.Load(AssetPath);` 其中Cls就是一个蓝图UClass；`const CMC = this.CharacterMovement;` ，CMC就是一个移动组件对象；
上一节中，我们可以知道，puerts加载了js主动获取的类，然后将其转为js 构造函数
我们知道，如果要在js中创建一个对象，就需要有对应构造函数，CMC对象是怎么来的？我们没有在js层构建一个CMC对象，而是从Character对象中获取的；以及我们并没有去主动从UE对象上获取`CharacterMovementComponent`类，这个类又是在什么时候加载到js的？
可是用户js代码中并没有直接去创建这个CMC，而是通过this.CharacterMovement来获取的，我们看看UE成员变量的获取的整个链路来解答这个问题

成员变量的getter/setter的设置，还是在上一节`UETypeToJsClass`中生成Js构造函数时做的，位置在`FStructWrapper::ToFunctionTemplate`中调用的`FStructWrapper::InitTemplateProperties`

简单讲一下`ToFunctionTemplate`，这个函数返回一个`v8::FunctionTemplate`，`FunctionTemplate`是用于生成Js Function的工厂模板；在调用方获取到FunctionTemplate后，可以FunctionTemplate->GetFunction(Context)，来创建一个Js Function；


通过迭代器TFieldIterator遍历这个UClass上的所有UProperty，为每一个UProperty生成一个FPropertyTranslator，并根据属性类型注册对应的访问器（`FPropertyTranslator::SetAccessor`），在PrototypeTemplate上注册 getter/setter；
PrototypeTemplate是FunctionTemplate的原型链模板；最后当我们get/set`this.CharacterMovement`时就会触发这个getter/setter

```cpp
auto GetterTemplate = v8::FunctionTemplate::New(Isolate, Getter, Self);  
auto SetterTemplate = v8::FunctionTemplate::New(Isolate, Setter, Self);
Template->PrototypeTemplate()->SetAccessorProperty(  
    FV8Utils::InternalString(Isolate, PropertyName), GetterTemplate, SetterTemplate, v8::DontDelete);
```
getter就是FPropertyTranslator::Getter，setter就是FPropertyTranslator::Setter




## 3. js侧创建UE对象

`const Vec = new UE.Vector(1,2,3);`

## 4. 委托回调函数

## 5. mixin（后续其它文档补充）
详见：
## 6. TS继承引擎类（后续其它文档补充）
详见：

# js与UE间数据传递 - 值，引用，指针

我们来看一个"bug"

```ts
const Montage = UE.AnimMontage.Load("Game/Animation/xxx.xxx")
const Track = Montage.SlotAnimTracks.Get(0);
const Sequence = UE.AnimSequence.Load("Game/Animation/Sequence.Sequence")
Track.AnimSegments.Set(0,Sequence);
// 这段代码执行完后，并没有在实际的Montage中修改掉第一段的Sequence
```
问题出在哪里？
`Track` 是一个UStruct，
```cpp
USTRUCT()  
struct FSlotAnimationTrack  
{  
    GENERATED_USTRUCT_BODY()  
    UPROPERTY(EditAnywhere, Category=Slot)  
    FName SlotName;  
    UPROPERTY(EditAnywhere, Category=Slot)  
    FAnimTrack AnimTrack;  
    ENGINE_API FSlotAnimationTrack();  
};
```
其中`AnimTrack` 也是一个UStruct，其中puerts对于UStruct的获取分为值拷贝和指针获取，对于UObject上的UPROPERTY()标注的UStruct是指针获取，对于UStruct上的UStruct，则是按值拷贝的；
```ts
Track.AnimSegments === Track.AnimSegments // false
```
这就是问题的所在了。解决办法的话基本上只有在`AnimMontage`上新加一个Extension暴露给ts ,比如`SetAnimReference`来更改AnimSequence；