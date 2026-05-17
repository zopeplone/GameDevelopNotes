深度魔改类似Angelscript-UE，将脚本能力嵌入UE引擎
# 魔改目标
- 不要像puerts的继承引擎类一样去生成蓝图中间层
- ts脚本可以写游戏运行逻辑和编辑器脚本，插件；可以在ts侧注册UClass，UStruct，UEnum
- 支持用tsx去写umg/slate
- 引擎内虚拟机唯一（类似Angelscript）
- 可以额外支持mixin，mixin生命周期需要和蓝图生命周期一起

1. 编写TsClass，自动生成UClass,UStruct
q: 如何注册脚本中的uclass?
a: 有两种办法
	第一种是像cocos一样，每次引擎启动时，自动运行这些脚本，然后通过装饰器来动态注册
	第二种是ts编译器去进行扫描代码AST，类似puerts自带的继承引擎类
脚本范例（我们希望以后的ts代码写成这样来注册到引擎中）
```ts
@ustruct()
export class FTsTestStruct {
	@uproperty(BlueprintReadWrite)
	Vec: FVector;
	@uproperty(BlueprintReadWrite)
	Quat: FQuat;
}

@uclass(Blueprintable)
export class ATsCharacterBase extends ACharacter {
	@uproperty(BlueprintReadWrite)
	TestNumberProp: number;
	@uproperty(BlueprintReadOnly)
	TestStringProp: string;
	@uproperty(BlueprintReadWrite)
	TestStruct: FTsTestStruct;
	// Actor组件
	@uproperty()
	TestComp: TsTestComp;
	Constructor(){
		this.CreateDefaultSubobject("TestComp", TsTestComp);
	}
	@ufunction(BlueprintCallable)
	EditStringProp(NewStr: string){
		TestStringProp = newStr;
	}
}

// 对于uenum注册,如果注册是运行脚本，则可以用registerUEnum，如果是编译器静态分析，则可以注释中加上 @uenum
/** @uenum **/
export enum ETestEnum {
	
}

// registerUEnum(ETestEnum)
```

2. UClass的构造函数

构造函数只在CDO创建时被执行一次！然后CDO一直存储在引擎和脚本中,
构造函数支持修改纯js变量
支持在一个Actor的构造函数修改Component的属性
构造函数和类不会跨虚拟机执行
```ts
Constructor(){
	const FpsCamera = this.CreateDefaultSubobject<UE.CameraComponent>("FpsCamera", UE.CameraComponent.StaticClass());  
	FpsCamera.SetupAttachment(this.CapsuleComponent, "FpsCamera");
	this.CreateDefaultSubobject("TestComp", TsTestComp);
	this.TestNumberProp = 12345;
	this.TsOnlyObj = {
		a:1,b:234,c:"asad"
	}
}
```

3. 包管理器与js存放位置
参考 [[puerts多模块Loader]]

4. js脚本初始化执行顺序
- 在ue初始化时，创建jsEnv，
- 执行用户侧的postInit.js(如果存在)
- 进行脚本中uclass等的注册，解析AST，发现有@uclass/ustruct/uenum则执行，如果有其它被引入的js，则会一并执行（对顶层逻辑不友好）
如果开始PIE游戏，则会执行用户侧的 main.js

5. 热重载
在编辑器中，有两种热重载，FullReload,HardReload(增量FullReload),SoftReload
- 在ue初始化时，FullReload，重新进行所有脚本uclass注册，构造所有CDO；
- 在编辑器打开时，进行的ts修改，会HardReload，执行逻辑和FullReload相同，但是只会进行增量的热重载，js的热重载，构造新的CDO；
- 在PIE时，进行SoftReload，只会进行js的热重载，不会修改ue的UClass等等，也不会构造CDO，会在PIE结束时再进行一次HardReload
# Puerts插件修改

1. 去掉继承引擎类的所有逻辑
2. 加入TsClass，TsStruct，TsEnum类，作为Ts生成出的类


