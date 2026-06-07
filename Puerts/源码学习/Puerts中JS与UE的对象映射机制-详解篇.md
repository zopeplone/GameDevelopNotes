# uelazyload

在puerts的`uelazyload.js` 中
```js
// registerBuildinModule 即为把'ue' 模块设置为UE这个Object
puerts.registerBuildinModule('ue', UE);
```
UE这个Object里面是什么呢?从这个`uelazyload`文件名就能推测出，是一个Proxy对象，用于从UE的类型系统里面获取到UClass反射类
### UE Object 构造
我们来看看这个Object是怎么构造出来的
1. 创建一个没有原型链的空对象，避免继承 Object.prototype上的属性干扰
```js
let UE = Object.create(null);
```
2. 创建Proxy拦截器，每次我们调用 UE.xxx 时，都会走这个Proxy的 get 方法
```js
let UE_proxy = new Proxy(UE, {
    get : function(UE, name) {
        let cls = loadUEType(name);    // 调用 C++ 函数加载类型
        rawSet(UE, name, cls);         // 缓存到 UE 对象上
        interceptClass(cls);           
        return cls;
    }
});
```
3. 将 Proxy 设为原型
```js
Object.setPrototypeOf(UE, UE_proxy);
```
**Proxy 被设为 `UE` 的原型**。当访问 `UE.XXX` 时：
- 先查 UE Object 自身的属性（已缓存的类型），命中则直接返回
- 未命中 → 沿原型链进入 Proxy 的 `get` 方法触发懒加载
这里有三个函数，**`loadUEType`**  `rawSet` `interceptClass`；
其中  `rawSet`就是一个defineProperty设置一个键和一个值，使其不可被枚举，enumerable: false 就是在 `for(const key in UE)` 时，不会把这些Class的key也查找出来
```js
 function rawSet(obj, key, val) {
   Object.defineProperty(obj, key, {
		value: val,
		writable: true,
		configurable: true,
		enumerable: false,
   });
}
```
`interceptClass` 用于解决UE 类型的属性名访问兼容性问题。UE 从 C++ 侧暴露的类，其属性以**FName 内部格式**存储为键名，而非普通 JS 字符串。该函数给类包一层 Proxy，使得 JS 代码可以用普通字符串名访问这些属性。
```js
function interceptClass(cls) {
	let cls_proxy = new Proxy(cls, {
		get : function(cls, name) {
			const fname = getFNameString(name);
			const p = Object.getOwnPropertyDescriptor(cls, fname);
			if (p) {
				Object.defineProperty(cls, name, p);
				return cls[fname];
			}
		}
	});
	Object.setPrototypeOf(cls_proxy, Object.getPrototypeOf(cls));
	Object.setPrototypeOf(cls, cls_proxy);
}
```

然后到了重头戏 `loadUEType`，
```js
let loadUEType = global.puerts.loadUEType;
```
### LoadUEType

loadUEType在执行uelazyload.js之前就已经准备好了，在`FJsEnvImpl`的构造函数中，对puerts Object设置了一个loadUEType的函数，其中绑定了FJsEnvImpl的LoadUEType
```cpp
 MethodBindingHelper<&FJsEnvImpl::LoadUEType>::Bind(Isolate, Context, PuertsObj, "loadUEType", This);
```
具体这些初始化流程解析放在后面讲
我们看 LoadUEType的函数声明，FunctionCallbackInfo就是js调用C++函数，C++用于获取  **js调用宿主函数上下文** 信息的参数
```cpp
void LoadUEType(const v8::FunctionCallbackInfo<v8::Value>& Info);
```
函数体有点长，我们来慢慢解析一下，先简单介绍一下v8的这些类型和术语，Isolate就是Js虚拟机，Context就是js环境（在chrome中，每个iframe都是独立的Context），用来进行环境隔离的，只需要知道我们的js都运行在Context下就好；
这三个Scope是v8的RAII守卫，先不用管
```cpp
v8::Isolate* Isolate = Info.GetIsolate();  
v8::Isolate::Scope IsolateScope(Isolate);  
v8::HandleScope HandleScope(Isolate);  
v8::Local<v8::Context> Context = Isolate->GetCurrentContext();  
v8::Context::Scope ContextScope(Context);
```
取js传递给宿主函数的第一个参数，为string，然后再转为FString
```cpp
const FString TypeName = FV8Utils::ToFString(Isolate, Info[0]);
```
然后通过这个TypeName来找对应的UClass，UScriptStruct，UEnum，
```cpp
UField* Type = FindAnyType<UClass>(TypeName);
if (!Type)
{
	Type = FindAnyType<UScriptStruct>(TypeName);
}
if (!Type)
{
	Type = FindAnyType<UEnum>(TypeName);
}
if (!Type)
{
	Type = LoadObject<UClass>(nullptr, *TypeName);
}
if (!Type)
{
	Type = LoadObject<UScriptStruct>(nullptr, *TypeName);
}
if (!Type)
{
	Type = LoadObject<UEnum>(nullptr, *TypeName);
}
```
FindAnyType就是UE的`FindFirstObject` 寻找类型系统中某个类里叫做这个名字的对象（以前版本用`FindObject<T>(ANY_PACKAGE, InShortName);`）

```cpp
return FindFirstObject<T>(  
    InShortName, EFindFirstObjectOptions::EnsureIfAmbiguous | EFindFirstObjectOptions::NativeFirst, ELogVerbosity::Error);
```

获取到类型系统中的类型了，调用`UETypeToJsClass`
```cpp
auto Result = UETypeToJsClass(Isolate, Context, Type);
```

UETypeToJsClass根据运行时类型分两路：

|                            |                                                                |
| -------------------------- | -------------------------------------------------------------- |
| **UClass / UScriptStruct** | 调用 GetJsClass(Struct, Context)) → 返回 v8 Function（相当于class构造函数） |
| **UEnum**                  | 手动创建普通 JS Object，填入 名↔值 双向映射                                   |
```cpp
if (const auto Struct = Cast<UStruct>(Type))  
{  
    return GetJsClass(Struct, Context);  
}
if (const auto Enum = Cast<UEnum>(Type)) //...
```
在GetJsClass中，调用GetTemplateInfoOfType 来 创建/缓存 v8 FunctionTemplate
```cpp
auto Ret = GetTemplateInfoOfType(InStruct, Existed)->Template.Get(MainIsolate)->GetFunction(Context).ToLocalChecked();
```
```cpp
struct FTemplateInfo
{
	v8::UniquePersistent<v8::FunctionTemplate> Template;
	std::shared_ptr<FStructWrapper> StructWrapper;
};
// 获取类型的V8模板信息，支持UStruct和UClass
FTemplateInfo* GetTemplateInfoOfType(UStruct* Class, bool& Existed);
```
v8 `FunctionTemplate`  是一个 **用于在 C++ 中“定义 JS 普通函数/类的工厂对象**，再`GetFunction`来创建一个真正的 JsFunction
在js中，class是用function+prototype来模拟出的
我们看看GetTemplateInfoOfType的逻辑，总之其返回了 `v8::FunctionTemplate` 供调用者来创建JsFunction给js侧使用

1. 查缓存 TypeToTemplateInfoMap `TMap<UStruct*, FTemplateInfo> TypeToTemplateInfoMap`
   ├── 命中 → 直接返回
   └── 未命中 ↓
2. GetStructWrapper(InStruct) → 获取/创建 FStructWrapper
   （TypeReflectionMap 缓存）
3. 如果是 UScriptStruct：
   ├── StructWrapper->ToFunctionTemplate(Isolate, FScriptStructWrapper)
   └── Inherit(父Struct的Template)
4. 如果是 UClass：
   ├── StructWrapper->ToFunctionTemplate (Isolate, FClassWrapper)
   ├── Inherit(父Class的Template)
   └── 继承父类的 IsNativeTakeJsRef 标记
5. 缓存到 TypeToTemplateInfoMap

### FStructWrapper
`StructWrapper`是 Puerts 中**承上启下的核心类**：它把 UE 反射系统的 UClass / UScriptStruct转换为`v8::FunctionTemplate`，并接到FJsEnvImpl的缓存中；
