## 学习 Google V8
本项目唯一的目的是帮助我学习 Google 的 V8 JavaScript 引擎。

### 目录
1. [简介](./notes/intro_cn.md)
2. [Address](#address)
3. [TaggedImpl](#taggedimpl)
4. [Object](#object)
5. [Handle](#handle)
6. [FunctionTemplate](#functiontemplate)
7. [ObjectTemplate](#objecttemplate)
8. [Small Integers](#small-integers)
9. [String 类型](./notes/string_cn.md)
10. [Roots](#roots)
11. [Heap](./notes/heap_cn.md)
12. [Builtins](#builtins)
13. [Compiler pipeline](#compiler-pipeline)
14. [CodeStubAssembler](#codestubassembler)
15. [Torque](#torque)
16. [WebAssembly](#webassembly)
17. [Promises](./notes/promises_cn.md)
18. [Snapshots](./notes/snapshots_cn.md)
19. [V8 构建产物](#v8-build-artifacts)
20. [V8 启动流程详解](#startup-walk-through)
21. [构建 V8](#building-v8)
22. [提交更改](#contributing-a-change)
23. [调试](#debugging)
24. [构建 Chromium](#building-chromium)
25. [Goma chromium](#goma)
26. [EcmaScript 笔记](./notes/ecmaspec_cn.md)
27. [GN 笔记](./notes/gn_cn.md)

### Isolate
Isolate 是 V8 运行时的一个独立副本，包含它自己的堆（Heap）。
两个不同的 Isolate 可以并行运行，并可被视为完全不同的沙箱化 V8 运行时实例。

### Context
为了允许独立的 JavaScript 应用程序在同一个 Isolate 中运行，必须为每个程序指定一个 Context。
这是为了避免它们相互干扰，例如通过更改提供的 Builtin 对象。

### Template
这是 ObjectTemplate 和 FunctionTemplate 的父类。请记住，在 JavaScript 中，函数可以像对象一样拥有字段。

```cpp
class V8_EXPORT Template : public Data {
 public:
  void Set(Local<Name> name, Local<Data> value,
           PropertyAttribute attributes = None);
  void SetPrivate(Local<Private> name, Local<Data> value,
                  PropertyAttribute attributes = None);
  V8_INLINE void Set(Isolate* isolate, const char* name, Local<Data> value);

  void SetAccessorProperty(
     Local<Name> name,
     Local<FunctionTemplate> getter = Local<FunctionTemplate>(),
     Local<FunctionTemplate> setter = Local<FunctionTemplate>(),
     PropertyAttribute attribute = None,
     AccessControl settings = DEFAULT);
```
`Set` 函数可用于在从此模板创建的实例上设置名称和值。
`SetAccessorProperty` 用于通过函数进行 get/set 的属性。

```cpp
enum PropertyAttribute {
  /** None. **/
  None = 0,
  /** ReadOnly, 即不可写。 **/
  ReadOnly = 1 << 0,
  /** DontEnum, 即不可枚举。 **/
  DontEnum = 1 << 1,
  /** DontDelete, 即不可配置。 **/
  DontDelete = 1 << 2
};

enum AccessControl {
  DEFAULT               = 0,
  ALL_CAN_READ          = 1,
  ALL_CAN_WRITE         = 1 << 1,
  PROHIBITS_OVERWRITING = 1 << 2
};
```

### ObjectTemplate
这些允许你在没有专用构造函数的情况下创建 JavaScript 对象。
当使用 ObjectTemplate 创建实例时，新实例将具有在 ObjectTemplate 上配置的属性和函数。

这类似于：
```js
const obj = {};
```
该类在 `include/v8.h` 中声明并继承自 Template：
```cpp
class V8_EXPORT ObjectTemplate : public Template { 
  ...
}
class V8_EXPORT Template : public Data {
  ...
}
class V8_EXPORT Data {
 private:                                                                       
  Data();                                                                       
};
```
我们创建一个 ObjectTemplate 实例，并可以向其添加属性，所有使用此 ObjectTemplate 实例创建的实例都将拥有这些属性。
这是通过调用 `Template` 类的成员函数 `Set` 来完成的。你需要为属性指定一个 `Local<Name>`。
`Name` 是 `Symbol` 和 `String` 的父类，两者都可以用作属性名称。

`Set` 的实现在 `src/api/api.cc` 中：
```cpp
void Template::Set(v8::Local<Name> name, v8::Local<Data> value, v8::PropertyAttribute attribute) {
  ...

  i::ApiNatives::AddDataProperty(isolate, templ, Utils::OpenHandle(*name),           
                                 value_obj,                                     
                                 static_cast<i::PropertyAttributes>(attribute));
}
```

在 [objecttemplate_test.cc](./test/objecttemplate_test.cc) 中有一个示例。

### FunctionTemplate
这是一个用于创建函数的模板，与 ObjectTemplate 一样，它继承自 Template：
```cpp
class V8_EXPORT FunctionTemplate : public Template {
}
```
请记住，JavaScript 中的函数可以像对象一样拥有属性。

在 [functiontemplate_test.cc](./test/functiontemplate_test.cc) 中有一个示例。

可以使用以下方式创建函数模板实例：
```cpp
  Local<FunctionTemplate> ft = FunctionTemplate::New(isolate_, function_callback, data);
  Local<Function> function = ft->GetFunction(context).ToLocalChecked();
```
函数可以使用以下方式调用：
```cpp
  MaybeLocal<Value> ret = function->Call(context, recv, 0, nullptr);
```
`Function::Call` 可以在 `src/api/api.cc` 中找到：
```cpp
  bool has_pending_exception = false;
  auto self = Utils::OpenHandle(this);                                               
  i::Handle<i::Object> recv_obj = Utils::OpenHandle(*recv);                          
  i::Handle<i::Object>* args = reinterpret_cast<i::Handle<i::Object>*>(argv);   
  Local<Value> result;                                                               
  has_pending_exception = !ToLocal<Value>(                                           
      i::Execution::Call(isolate, self, recv_obj, argc, args), &result);
```
注意 `Call` 的返回值是一个 `MaybeHandle<Object>`，它将被传递给 `ToLocal<Value>`，后者定义在 `api.h` 中：
```cpp
template <class T>                                                              
inline bool ToLocal(v8::internal::MaybeHandle<v8::internal::Object> maybe,      
                    Local<T>* local) {                                          
  v8::internal::Handle<v8::internal::Object> handle;                            
  if (maybe.ToHandle(&handle)) {                                                   
    *local = Utils::Convert<v8::internal::Object, T>(handle);                   
    return true;                                                                
  }                                                                                
  return false;                                                                 
}
```
现在让我们看看 `Execution::Call`，它位于 `execution/execution.cc` 中，并调用了：
```c++
return Invoke(isolate, InvokeParams::SetUpForCall(isolate, callable, receiver, argc, argv));
```
`SetUpForCall` 将返回一个 `InvokeParams`。
TODO：仔细查看 InvokeParams。
```c++
V8_WARN_UNUSED_RESULT MaybeHandle<Object> Invoke(Isolate* isolate,              
                                                 const InvokeParams& params) {
```
```c++
Handle<Object> receiver = params.is_construct                             
                                    ? isolate->factory()->the_hole_value()         
                                    : params.receiver; 
```
在我们的例子中，`is_construct` 为 false，因为我们没有使用 `new`，并且 receiver（函数中的 `this`）应该设置为我们传入的 receiver。之后我们有 `Builtins::InvokeApiFunction`：
```c++
auto value = Builtins::InvokeApiFunction(                                 
          isolate, params.is_construct, function, receiver, params.argc,        
          params.argv, Handle<HeapObject>::cast(params.new_target)); 
```

```c++
result = HandleApiCallHelper<false>(isolate, function, new_target,        
                                    fun_data, receiver, arguments);
```

`api-arguments-inl.h` 中有：
```c++
FunctionCallbackArguments::Call(CallHandlerInfo handler) {
  ...
  ExternalCallbackScope call_scope(isolate, FUNCTION_ADDR(f));                  
  FunctionCallbackInfo<v8::Value> info(values_, argv_, argc_);                  
  f(info);
  return GetReturnValue<Object>(isolate);
}
```
对 `f(info)` 的调用就是触发回调的地方，这只是一个普通的函数调用。

回到 `HandleApiCallHelper`：
```c++
Handle<Object> result = custom.Call(call_data);                             
                                                                                
RETURN_EXCEPTION_IF_SCHEDULED_EXCEPTION(isolate, Object);
```
`RETURN_EXCEPTION_IF_SCHEDULED_EXCEPTION` 展开为：
```c++
Handle<Object> result = custom.Call(call_data);                             
do { 
  Isolate* __isolate__ = (isolate); 
  ((void) 0); 
  if (__isolate__->has_scheduled_exception()) { 
    __isolate__->PromoteScheduledException(); 
    return MaybeHandle<Object>(); 
  }
} while (false);
```
注意，如果发生异常，将返回一个空对象。
稍后在 `execution.cc` 的 `Invoke` 中：
```c++
  auto value = Builtins::InvokeApiFunction(                                 
          isolate, params.is_construct, function, receiver, params.argc,        
          params.argv, Handle<HeapObject>::cast(params.new_target));            
  bool has_exception = value.is_null();                                     
  if (has_exception) {                                                      
    if (params.message_handling == Execution::MessageHandling::kReport) {   
    isolate->ReportPendingMessages();                                     
    }                                                                       
    return MaybeHandle<Object>();                                           
  } else {                                                                  
    isolate->clear_pending_message();                                       
  }                                                                         
  return value;                         
```
看起来返回一个空对象会导致触发异常？

### Address
`Address` 可以在 `include/v8-internal.h` 中找到：

```c++
typedef uintptr_t Address;
```
`uintptr_t` 是 `cstdint` 中指定的可选类型，能够存储数据指针。它是一个无符号整数类型，任何有效的 `void` 指针都可以转换为该类型（并转换回来）。

### TaggedImpl
该类在 `src/objects/tagged-impl.h` 中声明，并有一个私有成员，声明如下：
```c++
 public
  constexpr StorageType ptr() const { return ptr_; }
 private:
  StorageType ptr_;
```
可以使用以下方式创建实例：
```c++
  i::TaggedImpl<i::HeapObjectReferenceType::STRONG, i::Address>  tagged{};
```
存储类型也可以是 `Tagged_t`，它定义在 `globals.h` 中：
```c++
 using Tagged_t = uint32_t;
```
在使用指针压缩（Pointer Compression）时，它似乎可以是一个不同的值。

参见 [tagged_test.cc](./test/tagged_test.cc) 中的示例。

### Object
该类继承自 TaggedImpl：
```c++
class Object : public TaggedImpl<HeapObjectReferenceType::STRONG, Address> {       
```
可以使用默认构造函数创建 Object，或者通过传入一个 Address（这将委托给 TaggedImpl 构造函数）。
Object 本身没有任何成员（除了从 TaggedImpl 继承的 `ptr_`）。
因此，如果我们在栈上创建一个 Object，它就像是一个指向对象的指针/引用：
```
+------+
|Object|
|------|
|ptr_  |---->
+------+
```
现在，`ptr_` 是一个 StorageType，所以它可能是一个 `Smi`，在这种情况下它将直接包含值，例如一个小整数：
```
+------+
|Object|
|------|
|  18  |
+------+
```
参见 [object_test.cc](./test/object_test.cc) 中的示例。

### ObjectSlot
```c++
  i::Object obj{18};
  i::FullObjectSlot slot{&obj};
```

```
+----------+      +---------+
|ObjectSlot|      | Object  |
|----------|      |---------|
| address  | ---> |   18    |
+----------+      +---------+
```
参见 [objectslot_test.cc](./test/objectslot_test.cc) 中的示例。

### Maybe
Maybe 类似于 Optional，它可以持有一个值或为空。
```c++
template <class T>                                                              
class Maybe {
 public:
  V8_INLINE bool IsNothing() const { return !has_value_; }                      
  V8_INLINE bool IsJust() const { return has_value_; }
  ...

 private:
  bool has_value_;                                                              
  T value_; 
}
```
我起初认为 `Just` 这个名字有点令人困惑，但如果你这样读：
```c++
  bool cond = true;
  Maybe<int> maybe = cond ? Just<int>(10) : Nothing<int>();
```
我觉得它更有意义。有一些函数会检查 Maybe 是否为 Nothing，如果是则使进程崩溃。你也可以使用 `FromJust` 检查并返回值。

Maybe 的用法在于 API 调用可能会失败，返回 Nothing 是一种信号。

参见 [maybe_test.cc](./test/maybe_test.cc) 中的示例。

### MaybeLocal
```c++
template <class T>                                                              
class MaybeLocal {
 public:                                                                        
  V8_INLINE MaybeLocal() : val_(nullptr) {} 
  V8_INLINE Local<T> ToLocalChecked();
  V8_INLINE bool IsEmpty() const { return val_ == nullptr; }
  template <class S>                                                            
  V8_WARN_UNUSED_RESULT V8_INLINE bool ToLocal(Local<S>* out) const {           
    out->val_ = IsEmpty() ? nullptr : this->val_;                               
    return !IsEmpty();                                                          
  }    

 private:
  T* val_;
```
如果 `val_` 是 nullptr，`ToLocalChecked` 将使进程崩溃。如果你想避免崩溃，可以使用 `ToLocal`。

参见 [maybelocal_test.cc](./test/maybelocal_test.cc) 中的示例。

### Data
是 V8 堆中可以存在的所有对象的父类：
```c++
class V8_EXPORT Data {                                                          
 private:                                                                       
  Data();                                                                       
};
```

### Value
Value 继承自 `Data`，并添加了许多检查 Value 是否为特定类型的方法，如 `IsUndefined()`、`IsNull`、`IsNumber` 等。
它还有一些有用的方法可以转换为 Local<T>，例如：
```c++
V8_WARN_UNUSED_RESULT MaybeLocal<Number> ToNumber(Local<Context> context) const;
V8_WARN_UNUSED_RESULT MaybeLocal<String> ToNumber(Local<String> context) const;
...
```

### Handle
Handle 与 Object 和 ObjectSlot 类似，它也包含一个 Address 成员（在 `HandleBase` 中声明为 `location_`）。
不同之处在于它还包含一个 `location_` 成员（在 `HandleBase` 中声明），但其不同之处在于它是一个指向 Address 的指针。

### HandleScope
HandleScope 是一个栈分配的对象，它管理 Local 句柄。当一个 HandleScope 被销毁时，所有在其中创建的 Local 句柄都会被释放。

```c++
class V8_EXPORT HandleScope {
 public:
  explicit HandleScope(Isolate* isolate);
  ~HandleScope();
  ...
 private:
  internal::Isolate* isolate_;
  internal::Address* prev_next_;
  internal::Address* prev_limit_;
```
在 `src/api/api.cc` 中：
```c++
HandleScope::HandleScope(Isolate* v8_isolate) {
  i::Isolate* isolate = reinterpret_cast<i::Isolate*>(v8_isolate);
  isolate_ = isolate;
  i::HandleScopeData* data = isolate->handle_scope_data();
  prev_next_ = data->next;
  prev_limit_ = data->limit;
  data->level++;
}
```
`HandleScopeData` 是一个简单的结构体：
```c++
struct HandleScopeData {
  Address* next;
  Address* limit;
  int level;
  int sealed_level;
  CanonicalHandleScope* canonical_scope;

  void Initialize() {
    next = limit = nullptr;
    sealed_level = level = 0;
    canonical_scope = nullptr;
  }
};
```
注意有两个指针（Address*）指向 next 和 limit。当 HandleScope 初始化时，当前的 `handle_scope_data` 将从内部 Isolate 中检索。正在创建的 HandleScope 实例存储当前 Isolate 的 next/limit 指针，以便在关闭此 HandleScope 时恢复它们（参见 `CloseScope`）。

那么，创建了 HandleScope 后，Local<T> 如何与此实例交互？

当创建一个 Local<T> 时，它可能会经过 `FactoryBase::NewStruct`，该函数将分配一个新的 Map，然后为正在创建的 InstanceType 创建一个 Handle：
```c++
Handle<Struct> str = handle(Struct::cast(result), isolate()); 
```
这将进入 `src/handles/handles-inl.h` 中的 `Handle<T>` 构造函数：
```c++
template <typename T>                                                           
Handle<T>::Handle(T object, Isolate* isolate): HandleBase(object.ptr(), isolate) {}

HandleBase::HandleBase(Address object, Isolate* isolate)                        
    : location_(HandleScope::GetHandle(isolate, object)) {}
```
注意使用 `object.ptr()` 将 Address 传递给 HandleBase。
还要注意 HandleBase 将其 `location_` 设置为 `HandleScope::GetHandle` 的结果。

```c++
Address* HandleScope::GetHandle(Isolate* isolate, Address value) {              
  DCHECK(AllowHandleAllocation::IsAllowed());                                   
  HandleScopeData* data = isolate->handle_scope_data();                         
  CanonicalHandleScope* canonical = data->canonical_scope;                      
  return canonical ? canonical->Lookup(value) : CreateHandle(isolate, value);   
}
```
在这种情况下将调用 `CreateHandle`，该函数将检索当前 Isolate 的 `handle_scope_data`：
```c++
  HandleScopeData* data = isolate->handle_scope_data();                         
  Address* result = data->next;                                                 
  if (result == data->limit) {                                                  
    result = Extend(isolate);                                                   
  }     
```
在这种情况下，next 和 limit 都将是 0x0，因此将调用 `Extend`。
`Extend` 也会获取 Isolate 的 `handle_scope_data` 并检查当前级别，之后获取 Isolate 的 `HandleScopeImplementer`：
```c++
  HandleScopeImplementer* impl = isolate->handle_scope_implementer();           
```
`HandleScopeImplementer` 在 `src/api/api.h` 中声明。

`HandleScope::CreateHandle` 将从 Isolate 获取 `handle_scope_data`：
```c++
Address* HandleScope::CreateHandle(Isolate* isolate, Address value) {
  HandleScopeData* data = isolate->handle_scope_data();
  if (result == data->limit) {
    result = Extend(isolate);
  }
  // 更新当前的 next 字段，在创建的句柄中设置值，并返回结果。
  data->next = reinterpret_cast<Address*>(reinterpret_cast<Address>(result) + sizeof(Address));
  *result = value;
  return result;
}                         
```
注意 `data->next` 被设置为传入的地址 + Address 的大小。

HandleScope 的析构函数将调用 `CloseScope`。
参见 [handlescope_test.cc](./test/handlescope_test.cc) 中的示例。

### EscapableHandleScope
Local 句柄位于栈上，并在调用相应的析构函数时被删除。如果存在局部的 HandleScope，那么它将在作用域返回时处理此问题。当句柄不再有引用时，它可以被垃圾回收。这意味着如果一个函数有一个 HandleScope 并希望返回一个 handle/local，那么在函数返回后它将不可用。这就是 EscapableHandleScope 的用途，它允许将值放置在封闭的句柄作用域中以使其存活。当封闭的 HandleScope 超出作用域时，它将被清理。

```c++
class V8_EXPORT EscapableHandleScope : public HandleScope {                        
 public:                                                                           
  explicit EscapableHandleScope(Isolate* isolate);
  V8_INLINE ~EscapableHandleScope() = default;
  template <class T>
  V8_INLINE Local<T> Escape(Local<T> value) {
    internal::Address* slot = Escape(reinterpret_cast<internal::Address*>(*value));
    return Local<T>(reinterpret_cast<T*>(slot));
  }

  template <class T>
  V8_INLINE MaybeLocal<T> EscapeMaybe(MaybeLocal<T> value) {
    return Escape(value.FromMaybe(Local<T>()));
  }

 private:
  ...
  internal::Address* escape_slot_;
};
```

来自 `api.cc`：
```c++
EscapableHandleScope::EscapableHandleScope(Isolate* v8_isolate) {
  i::Isolate* isolate = reinterpret_cast<i::Isolate*>(v8_isolate);
  escape_slot_ = CreateHandle(isolate, i::ReadOnlyRoots(isolate).the_hole_value().ptr());
  Initialize(v8_isolate);
}
```
因此，当创建一个 EscapableHandleScope 时，它将使用 hole value 创建一个句柄，并将其存储在类型为 Address 的 `escape_slot_` 中。这个 Handle 将在当前的 HandleScope 中创建，EscapableHandleScope 稍后可以为该指针/地址设置它想要逃逸（escape）的值。稍后当该 HandleScope 超出作用域时，它将被清理。然后它像普通的 HandleScope 一样调用 `Initialize`。

```c++
i::Address* HandleScope::CreateHandle(i::Isolate* isolate, i::Address value) {
  return i::HandleScope::CreateHandle(isolate, value);
}
```
来自 `handles-inl.h`：
```c++
Address* HandleScope::CreateHandle(Isolate* isolate, Address value) {
  DCHECK(AllowHandleAllocation::IsAllowed());
  HandleScopeData* data = isolate->handle_scope_data();
  Address* result = data->next;
  if (result == data->limit) {
    result = Extend(isolate);
  }
  // 更新当前的 next 字段，在创建的句柄中设置值，并返回结果。
  DCHECK_LT(reinterpret_cast<Address>(result),
            reinterpret_cast<Address>(data->limit));
  data->next = reinterpret_cast<Address*>(reinterpret_cast<Address>(result) +
                                          sizeof(Address));
  *result = value;
  return result;
}
```

当调用 `Escape` 时，会发生以下情况（`v8.h`）：
```c++
template <class T>
  V8_INLINE Local<T> Escape(Local<T> value) {
    internal::Address* slot = Escape(reinterpret_cast<internal::Address*>(*value));
    return Local<T>(reinterpret_cast<T*>(slot));
  }
```
以及 `EscapableHandleScope::Escape`（`api.cc`）：
```c++
i::Address* EscapableHandleScope::Escape(i::Address* escape_value) {
  i::Heap* heap = reinterpret_cast<i::Isolate*>(GetIsolate())->heap();
  Utils::ApiCheck(i::Object(*escape_slot_).IsTheHole(heap->isolate()),
                  "EscapableHandleScope::Escape", "Escape value set twice");
  if (escape_value == nullptr) {
    *escape_slot_ = i::ReadOnlyRoots(heap).undefined_value().ptr();
    return nullptr;
  }
  *escape_slot_ = *escape_value;
  return escape_slot_;
}
```
如果 `escape_value` 为空，则将指向父 HandleScope 的 `escape_slot` 设置为 `undefined_value()` 而不是之前的 hole value，并返回 nullptr。这个返回的地址/指针在转换为 T* 后被返回。
接下来，我们看看当 EscapableHandleScope 超出作用域时会发生什么。这将调用 `HandleScope::~HandleScope`，这是合理的，因为任何其他 Local 句柄都应该被清理。

`Escape` 将其参数的值复制到封闭的作用域中，删除其所有的局部句柄，然后返回可以安全返回的新句柄副本。

### HeapObject
TODO:

### Local
只有一个成员 `val_`，其类型为指向 `T` 的指针：
```c++
template <class T> class Local { 
...
 private:
  T* val_
}
```
注意这是一个指向 T 的指针。我们可以使用以下方式创建一个 local：
```c++
  v8::Local<v8::Value> empty_value;
```

因此，Local 包含一个指向类型 T 的指针。我们可以使用 `operator->` 和 `operator*` 访问此指针。

我们可以使用 `Local::Cast` 从子类型转换为父类型：
```c++
v8::Local<v8::Number> nr = v8::Local<v8::Number>(v8::Number::New(isolate_, 12));
v8::Local<v8::Value> val = v8::Local<v8::Value>::Cast(nr);
```
还有：
```c++
v8::Local<v8::Value> val2 = nr.As<v8::Value>();
```

参见 [local_test.cc](./test/local_test.cc) 中的示例。

### PrintObject
从 C++ 使用 `_v8_internal_Print_Object`：
```console
$ nm -C libv8_monolith.a | grep Print_Object
0000000000000000 T _v8_internal_Print_Object(void*)
```
注意此函数没有命名空间。
我们可以这样使用它：
```c++
extern void _v8_internal_Print_Object(void* object);

_v8_internal_Print_Object(*((v8::internal::Object**)(*global)));
```
让我们仔细看看上面的代码：
```c++
  v8::internal::Object** gl = ((v8::internal::Object**)(*global));
```
我们使用解引用运算符获取 Local 的值（`*global`），它只是 `T*` 类型，即指向 Local 类型的指针：
```c++
template <class T>
class Local {
  ...
 private:
  T* val_;
}
```

然后我们将其转换为指向 Object 的指针的指针类型。
```
  gl**        Object*         Object
+-----+      +------+      +-------+
|     |----->|      |----->|       |
+-----+      +------+      +-------+
```
`v8::internal::Object` 实例只有一个数据成员，即名为 `ptr_` 的 `Address` 类型字段：

`src/objects/objects.h`：
```c++
class Object : public TaggedImpl<HeapObjectReferenceType::STRONG, Address> {
 public:
  constexpr Object() : TaggedImpl(kNullAddress) {}
  explicit constexpr Object(Address ptr) : TaggedImpl(ptr) {}

#define IS_TYPE_FUNCTION_DECL(Type) \
  V8_INLINE bool Is##Type() const;  \
  V8_INLINE bool Is##Type(const Isolate* isolate) const;
  OBJECT_TYPE_LIST(IS_TYPE_FUNCTION_DECL)
  HEAP_OBJECT_TYPE_LIST(IS_TYPE_FUNCTION_DECL)
  IS_TYPE_FUNCTION_DECL(HashTableBase)
  IS_TYPE_FUNCTION_DECL(SmallOrderedHashTable)
#undef IS_TYPE_FUNCTION_DECL
  V8_INLINE bool IsNumber(ReadOnlyRoots roots) const;
}
```
让我们看看其中一个函数是如何实现的。例如，在 `OBJECT_TYPE_LIST` 中我们有：
```c++
#define OBJECT_TYPE_LIST(V) \
  V(LayoutDescriptor)       \
  V(Primitive)              \
  V(Number)                 \
  V(Numeric)
```
因此，Object 类将有一个类似于以下的函数：
```c++
inline bool IsNumber() const;
inline bool IsNumber(const Isolate* isolate) const;
```
在 `src/objects/objects-inl.h` 中我们将有实现：
```c++
bool Object::IsNumber() const {
  return IsHeapObject() && HeapObject::cast(*this).IsNumber();
}
```
`IsHeapObject` 定义在 TaggedImpl 中：
```c++
  constexpr inline bool IsHeapObject() const { return IsStrong(); }

  constexpr inline bool IsStrong() const {
#if V8_HAS_CXX14_CONSTEXPR
    DCHECK_IMPLIES(!kCanBeWeak, !IsSmi() == HAS_STRONG_HEAP_OBJECT_TAG(ptr_));
#endif
    return kCanBeWeak ? HAS_STRONG_HEAP_OBJECT_TAG(ptr_) : !IsSmi();
  }
```

宏可以在 `src/common/globals.h` 中找到：
```c++
#define HAS_STRONG_HEAP_OBJECT_TAG(value)                          \
  (((static_cast<i::Tagged_t>(value) & ::i::kHeapObjectTagMask) == \
    ::i::kHeapObjectTag))
```
因此，我们将 `ptr_`（Address 类型）转换为 `Tagged_t` 类型，该类型定义在 `src/common/global.h` 中，根据是否使用压缩指针，它可能会有所不同。如果不支持压缩指针，它与 Address 相同：
``` 
using Tagged_t = Address;
```

`src/objects/tagged-impl.h`：
```c++
template <HeapObjectReferenceType kRefType, typename StorageType>
class TaggedImpl {

  StorageType ptr_;
}
```
`HeapObjectReferenceType` 可以是 WEAK 或 STRONG。在这种情况下，存储类型是 `Address`。因此，Object 本身只有一个从其唯一的父类继承的成员，即 `ptr_`。

因此，以下代码是告诉编译器将我们的 Local 的值 `*global` 视为一个指针（它已经是一个指针），指向一个指向符合 `v8::internal::Object` 类型布局的内存位置的指针，我们现在知道该类型有一个 `ptr_` 成员。我们想要解引用它并将其传递给函数。
```c++
_v8_internal_Print_Object(*((v8::internal::Object**)(*global)));
```

### ObjectTemplate
但我仍然缺少 ObjectTemplate 和 Object 之间的联系。
当我们创建它时，我们使用：
```c++
Local<ObjectTemplate> global = ObjectTemplate::New(isolate);
```
在 `src/api/api.cc` 中我们有：
```c++
static Local<ObjectTemplate> ObjectTemplateNew(
    i::Isolate* isolate, v8::Local<FunctionTemplate> constructor,
    bool do_not_cache) {
  i::Handle<i::Struct> struct_obj = isolate->factory()->NewStruct(
      i::OBJECT_TEMPLATE_INFO_TYPE, i::AllocationType::kOld);
  i::Handle<i::ObjectTemplateInfo> obj = i::Handle<i::ObjectTemplateInfo>::cast(struct_obj);
  InitializeTemplate(obj, Consts::OBJECT_TEMPLATE);
  int next_serial_number = 0;
  if (!constructor.IsEmpty())
    obj->set_constructor(*Utils::OpenHandle(*constructor));
  obj->set_data(i::Smi::zero());
  return Utils::ToLocal(obj);
}
```
在这种情况下，`Struct` 是什么？
`src/objects/struct.h`
```c++
#include "torque-generated/class-definitions-tq.h"

class Struct : public TorqueGeneratedStruct<Struct, HeapObject> {
 public:
  inline void InitializeBody(int object_size);
  void BriefPrintDetails(std::ostream& os);
  TQ_OBJECT_CONSTRUCTORS(Struct)
```
注意 include 指定了 `torque-generated` 的头文件，可以在 `out/x64.release_gcc/gen/torque-generated/class-definitions-tq` 中找到。因此，在某个地方一定有对 `torque` 可执行文件的调用，它在编译主源文件之前生成 Code Stub Assembler 的 C++ 头文件和源文件。在 `Building V8` 章节中有一节关于此内容。
宏 `TQ_OBJECT_CONSTRUCTORS` 可以在 `src/objects/object-macros.h` 中找到，并展开为：
```c++
  constexpr Struct() = default;

 protected:
  template <typename TFieldType, int kFieldOffset>
  friend class TaggedField;

  inline explicit Struct(Address ptr);
```

那么 TorqueGeneratedStruct 看起来像什么？
```
template <class D, class P>
class TorqueGeneratedStruct : public P {
 public:
```
在这种情况下，D 是 Struct，P 是 HeapObject。但上面是类型的声明，我们在 .h 文件中拥有的是生成的内容。

此类型定义在 `src/objects/struct.tq` 中：
```
@abstract                                                                       
@generatePrint                                                                  
@generateCppClass                                                               
extern class Struct extends HeapObject {
}
```
这看起来像是一种领域特定语言（DSL），用于定义 V8 中的对象布局。

### 线程池大小
```c++
V8::SetThreadPoolSize(7);
```
在 `src/api/api.cc` 中：
```c++
void V8::SetThreadPoolSize(int thread_pool_size) {
  i::V8::set_thread_pool_size(thread_pool_size);
}
```
在 `src/init/v8.cc` 中：
```c++
void V8::set_thread_pool_size(int thread_pool_size) {
  if (thread_pool_size > 0) {
    thread_pool_size_ = std::min(thread_pool_size, kMaxThreadPoolSize);
  }
}
```
`kMaxThreadPoolSize` 定义在 `src/init/v8.h` 中：
```c++
static const int kMaxThreadPoolSize = 16;
```
因此，线程池大小被限制在 1 到 16 之间。

### 平台初始化
```c++
v8::Platform* platform = v8::platform::NewDefaultPlatform().release();
v8::V8::InitializePlatform(platform);
```
在 `src/api/api.cc` 中：
```c++
void V8::InitializePlatform(Platform* platform) {
  i::V8::InitializePlatform(platform);
}
```
在 `src/init/v8.cc` 中：
```c++
void V8::InitializePlatform(v8::Platform* platform) {
  CHECK_NULL(platform_);
  platform_ = platform;
}
```
`platform_` 是 `i::V8` 类的静态成员。

### V8 初始化
```c++
v8::V8::Initialize();
```
在 `src/api/api.cc` 中：
```c++
bool V8::Initialize() {
  i::V8::Initialize();
  return true;
}
```
在 `src/init/v8.cc` 中：
```c++
void V8::Initialize() {
  if (is_initialized_) return;
  is_initialized_ = true;
  ...
}
```
`is_initialized_` 也是 `i::V8` 类的静态成员。

### ICU 初始化
```c++
v8::V8::InitializeICU();
```
ICU 代表 International Components for Unicode。它用于国际化支持。

### 外部启动数据
```c++
v8::V8::InitializeExternalStartupData(argv[0]);
```
这用于加载快照文件（snapshot files）。快照用于通过预先反序列化堆的状态来加速 V8 的启动。

### 线程池大小（续）
```c++
(int) thread_pool_size = 7
```
这就是 `SetThreadPoolSize` 所做的一切。在此之后我们有：
```c++
    platform->EnsureInitialized();

    for (int i = 0; i < thread_pool_size_; ++i)
      thread_pool_.push_back(new WorkerThread(&queue_));
```
`new WorkerThread` 将创建一个新的 pthread（在我的系统上，即 MacOSX）：
```c++
    result = pthread_create(&data_->thread_, &attr, ThreadEntry, this);
```
`ThreadEntry` 可以在 `src/base/platform/platform-posix` 中找到。

### International Component for Unicode (ICU)
International Components for Unicode (ICU) 处理国际化（i18n）。
ICU 提供对本地化敏感的字符串比较、日期/时间/数字/货币格式化等支持。

有一个名为 ECMAScript 402 的可选 API，V8 支持它且默认启用。[i18n-support](https://github.com/v8/v8/wiki/i18n-support) 指出，即使你的应用程序不使用 ICU，你仍然需要调用 `InitializeICU`：
```c++
    V8::InitializeICU();
```

### Local
```c++
Local<String> script_name = ...;
```
那么 `script_name` 是什么？它是一个由 V8 GC 管理的对象引用。
GC 需要能够移动对象（移动指针）并跟踪对象是否应该被回收。
与持久句柄（Persistent handles）相比，Local 句柄是轻量级的，主要用于局部操作。
这些句柄由 HandleScope 管理，因此你必须在栈上有一个 HandleScope，且 Local 仅在 HandleScope 有效期间有效。
这使用了资源获取即初始化（RAII）模式，因此当 HandleScope 实例超出作用域时，它将移除所有的 Local 实例。

`Local` 类（在 `include/v8.h` 中）只有一个成员，其类型为指向类型 `T` 的指针。对于上述示例，它将是：
```c++
  String* val_;
```
你可以在 `include/v8.h` 中找到 Local 的可用操作。

```shell
(lldb) p script_name.IsEmpty()
(bool) $12 = false
```

Local<T> 重载了许多运算符，例如 `->`：
```shell
(lldb) p script_name->Length()
(int) $14 = 7
```
其中 `Length` 是 V8 String 类的一个方法。

句柄栈（Handle stack）不是 C++ 调用栈的一部分，但 HandleScope 嵌入在 C++ 栈中。HandleScope 只能在栈上分配，不能使用 `new` 分配。

### Persistent
https://v8.dev/docs/embed:
持久句柄提供对堆分配的 JavaScript 对象的引用，就像 Local 句柄一样。有两种类型，它们在所处理引用的生命周期管理上有所不同。当你需要在多个函数调用中保留对对象的引用，或者当句柄生命周期与 C++ 作用域不对应时，请使用持久句柄。例如，Google Chrome 使用持久句柄来引用文档对象模型（DOM）节点。

可以使用 `PersistentBase::SetWeak` 将持久句柄设为弱引用，以便当对象的唯一引用来自弱持久句柄时，触发垃圾回收器的回调。

`UniquePersistent<SomeType>` 句柄依赖 C++ 构造函数和析构函数来管理底层对象的生命周期。
`Persistent<SomeType>` 可以通过其构造函数构造，但必须使用 `Persistent::Reset` 显式清除。

那么持久对象是如何创建的呢？
让我们写一个测试来找出答案（`test/persistent-object_text.cc`）：
```console
$ make test/persistent-object_test
$ ./test/persistent-object_test --gtest_filter=PersistentTest.value
```
现在，要创建一个 Persistent 实例，我们需要一个 Local<T> 实例，否则 Persistent 实例将为空。
```c++
Local<Object> o = Local<Object>::New(isolate_, Object::New(isolate_));
```
`Local<Object>::New` 可以在 `src/api/api.cc` 中找到：
```c++
Local<v8::Object> v8::Object::New(Isolate* isolate) {
  i::Isolate* i_isolate = reinterpret_cast<i::Isolate*>(isolate);
  LOG_API(i_isolate, Object, New);
  ENTER_V8_NO_SCRIPT_NO_EXCEPTION(i_isolate);
  i::Handle<i::JSObject> obj =
      i_isolate->factory()->NewJSObject(i_isolate->object_function());
  return Utils::ToLocal(obj);
}
```
首先发生的是将公共的 Isolate 指针转换为指向内部 `Isolate` 类型的指针。
`LOG_API` 是同一源文件（`src/api/api.cc`）中的一个宏：
```c++
#define LOG_API(isolate, class_name, function_name)                           \
  i::RuntimeCallTimerScope _runtime_timer(                                    \
      isolate, i::RuntimeCallCounterId::kAPI_##class_name##_##function_name); \
  LOG(isolate, ApiEntryCall("v8::" #class_name "::" #function_name))
```
在我们的例子中，预处理器会将其展开为：
```c++
  i::RuntimeCallTimerScope _runtime_timer(
      isolate, i::RuntimeCallCounterId::kAPI_Object_New);
  LOG(isolate, ApiEntryCall("v8::Object::New))
```
`LOG` 是一个可以在 `src/log.h` 中找到的宏：
```c++
#define LOG(isolate, Call)                              \
  do {                                                  \
    v8::internal::Logger* logger = (isolate)->logger(); \
    if (logger->is_logging()) logger->Call;             \
  } while (false)
```
这将展开为：
```c++
  v8::internal::Logger* logger = isolate->logger();
  if (logger->is_logging()) logger->ApiEntryCall("v8::Object::New");
```
展开 LOG_API 宏后，我们得到：
```c++
Local<v8::Object> v8::Object::New(Isolate* isolate) {
  i::Isolate* i_isolate = reinterpret_cast<i::Isolate*>(isolate);
  i::RuntimeCallTimerScope _runtime_timer( isolate, i::RuntimeCallCounterId::kAPI_Object_New);
  v8::internal::Logger* logger = isolate->logger();
  if (logger->is_logging()) logger->ApiEntryCall("v8::Object::New");

  ENTER_V8_NO_SCRIPT_NO_EXCEPTION(i_isolate);
  i::Handle<i::JSObject> obj =
      i_isolate->factory()->NewJSObject(i_isolate->object_function());
  return Utils::ToLocal(obj);
}
```
接下来是 `ENTER_V8_NO_SCRIPT_NO_EXCEPTION`：
```c++
#define ENTER_V8_NO_SCRIPT_NO_EXCEPTION(isolate)                    \
  i::VMState<v8::OTHER> __state__((isolate));                       \
  i::DisallowJavascriptExecutionDebugOnly __no_script__((isolate)); \
  i::DisallowExceptions __no_exceptions__((isolate))
```
展开宏后：
```c++
Local<v8::Object> v8::Object::New(Isolate* isolate) {
  i::Isolate* i_isolate = reinterpret_cast<i::Isolate*>(isolate);
  i::RuntimeCallTimerScope _runtime_timer( isolate, i::RuntimeCallCounterId::kAPI_Object_New);
  v8::internal::Logger* logger = isolate->logger();
  if (logger->is_logging()) logger->ApiEntryCall("v8::Object::New");

  i::VMState<v8::OTHER> __state__(i_isolate));
  i::DisallowJavascriptExecutionDebugOnly __no_script__(i_isolate);
  i::DisallowExceptions __no_exceptions__(i_isolate));

  i::Handle<i::JSObject> obj =
      i_isolate->factory()->NewJSObject(i_isolate->object_function());

  return Utils::ToLocal(obj);
}
```
TODO：仔细查看 `VMState`。

首先，调用 `i_isolate->object_function()` 并将结果传递给 `NewJSObject`。`object_function` 是由名为 `NATIVE_CONTEXT_FIELDS` 的宏生成的：
```c++
#define NATIVE_CONTEXT_FIELD_ACCESSOR(index, type, name)     \
  Handle<type> Isolate::name() {                             \
    return Handle<type>(raw_native_context()->name(), this); \
  }                                                          \
  bool Isolate::is_##name(type* value) {                     \
    return raw_native_context()->is_##name(value);           \
  }
NATIVE_CONTEXT_FIELDS(NATIVE_CONTEXT_FIELD_ACCESSOR)
```
`NATIVE_CONTEXT_FIELDS` 是 `src/contexts` 中的一个宏：
```c++
#define NATIVE_CONTEXT_FIELDS(V)                                               \
...                                                                            \
  V(OBJECT_FUNCTION_INDEX, JSFunction, object_function)                        \
```

```c++
  Handle<type> Isolate::object_function() {
    return Handle<JSFunction>(raw_native_context()->object_function(), this);
  }

  bool Isolate::is_object_function(JSFunction* value) {
    return raw_native_context()->is_object_function(value);
  }
```
我不清楚不同类型的 Context，有 Native Context，还有“普通/公共”Context。
在 `src/contexts-inl.h` 中，我们有 `native_context` 函数：
```c++
Context* Context::native_context() const {
  Object* result = get(NATIVE_CONTEXT_INDEX);
  DCHECK(IsBootstrappingOrNativeContext(this->GetIsolate(), result));
  return reinterpret_cast<Context*>(result);
}
```
`Context` 继承自 `FixedArray`，因此 `get` 函数是 `FixedArray` 的 `get` 函数，而 `NATIVE_CONTEXT_INDEX` 是数组中存储 Native Context 的索引。

现在，让我们仔细看看 `NewJSObject`。如果你在 `src/heap/factory.cc` 中搜索 `NewJSObject`：
```c++
Handle<JSObject> Factory::NewJSObject(Handle<JSFunction> constructor, PretenureFlag pretenure) {
  JSFunction::EnsureHasInitialMap(constructor);
  Handle<Map> map(constructor->initial_map(), isolate());
  return NewJSObjectFromMap(map, pretenure);
}
```
`NewJSObjectFromMap`：
```c++
...
  HeapObject* obj = AllocateRawWithAllocationSite(map, pretenure, allocation_site);
```
所以我们创建了一个新的 Map。

### Map
HeapObject 包含一个指向 Map 的指针，或者更确切地说，有一个返回指向 Map 的指针的函数。我在 HeapObject 类中看不到任何成员 map。

让我们看看 Map 是何时创建的。
```console
(lldb) br s -f map_test.cc -l 63
```

```c++
Handle<Map> Factory::NewMap(InstanceType type,
                            int instance_size,
                            ElementsKind elements_kind,
                            int inobject_properties) {
  HeapObject* result = isolate()->heap()->AllocateRawWithRetryOrFail(Map::kSize, MAP_SPACE);
  result->set_map_after_allocation(*meta_map(), SKIP_WRITE_BARRIER);
  return handle(InitializeMap(Map::cast(result), type, instance_size,
                              elements_kind, inobject_properties),
                isolate());
}
```
我们可以看到上面调用了堆实例上的 `AllocateRawWithRetryOrFail`，传入大小 `88` 并指定 `MAP_SPACE`：
```c++
HeapObject* Heap::AllocateRawWithRetryOrFail(int size, AllocationSpace space,
                                             AllocationAlignment alignment) {
  AllocationResult alloc;
  HeapObject* result = AllocateRawWithLigthRetry(size, space, alignment);
  if (result) return result;

  isolate()->counters()->gc_last_resort_from_handles()->Increment();
  CollectAllAvailableGarbage(GarbageCollectionReason::kLastResort);
  {
    AlwaysAllocateScope scope(isolate());
    alloc = AllocateRaw(size, space, alignment);
  }
  if (alloc.To(&result)) {
    DCHECK(result != exception());
    return result;
  }
  // TODO(1181417): Fix this.
  FatalProcessOutOfMemory("CALL_AND_RETRY_LAST");
  return nullptr;
}
```
`alignment` 的默认值是 `kWordAligned`。阅读头文件中的文档，它说该函数将尝试在 `MAP_SPACE` 中执行大小为 `88` 的分配，如果失败，将执行完整的 GC 并重试分配。
让我们看看 `AllocateRawWithLigthRetry`：
```c++
  AllocationResult alloc = AllocateRaw(size, space, alignment);
```
`AllocateRaw` 可以在 `src/heap/heap-inl.h` 中找到。根据 `space` 参数，会采取不同的路径。由于在我们的例子中是 `MAP_SPACE`，我们将关注该路径：
```c++
AllocationResult Heap::AllocateRaw(int size_in_bytes, AllocationSpace space, AllocationAlignment alignment) {
  ...
  HeapObject* object = nullptr;
  AllocationResult allocation;
  if (OLD_SPACE == space) {
  ...
  } else if (MAP_SPACE == space) {
    allocation = map_space_->AllocateRawUnaligned(size_in_bytes);
  }
  ...
}
```
`map_space_` 是 Heap 的私有成员（`src/heap/heap.h`）：
```c++
MapSpace* map_space_;
```
`AllocateRawUnaligned` 可以在 `src/heap/spaces-inl.h` 中找到：
```c++
AllocationResult PagedSpace::AllocateRawUnaligned( int size_in_bytes, UpdateSkipList update_skip_list) {
  if (!EnsureLinearAllocationArea(size_in_bytes)) {
    return AllocationResult::Retry(identity());
  }

  HeapObject* object = AllocateLinearly(size_in_bytes);
  MSAN_ALLOCATED_UNINITIALIZED_MEMORY(object->address(), size_in_bytes);
  return object;
}
```
`update_skip_list` 的默认值是 `UPDATE_SKIP_LIST`。
让我们看看 `AllocateLinearly`：
```c++
HeapObject* PagedSpace::AllocateLinearly(int size_in_bytes) {
  Address current_top = allocation_info_.top();
  Address new_top = current_top + size_in_bytes;
  allocation_info_.set_top(new_top);
  return HeapObject::FromAddress(current_top);
}
```
回想一下，在我们的例子中 `size_in_bytes` 是 `88`。
```console
(lldb) expr current_top
(v8::internal::Address) $5 = 24847457492680
(lldb) expr new_top
(v8::internal::Address) $6 = 24847457492768
(lldb) expr new_top - current_top
(unsigned long) $7 = 88
```
注意，首先将 top 设置为 new_top，然后返回 current_top，这将是一个指向内存中对象起始位置的指针（在这种情况下是 `v8::internal::Map`，它也是 HeapObject 类型）。
我一直在想为什么 Map（和其他 HeapObject）没有任何成员字段，而只有/主要是各种组成对象的字段的 getter/setter。答案是，指向例如 Map 实例的指针指向实例的第一个内存位置。而 getter/setter 函数使用索引来读写内存位置。索引主要以枚举字段的形式存在，定义了类型的内存布局。

接下来，在 `AllocateRawUnaligned` 中我们有 `MSAN_ALLOCATED_UNINITIALIZED_MEMORY` 宏：
```c++
  MSAN_ALLOCATED_UNINITIALIZED_MEMORY(object->address(), size_in_bytes);
```
`MSAN_ALLOCATED_UNINITIALIZED_MEMORY` 可以在 `src/msan.h` 中找到，`ms` 代表 `Memory Sanitizer`，仅在定义了 `V8_US_MEMORY_SANITIZER` 时使用。
返回的 `object` 在返回时将用于构造 `AllocationResult`。
回到 `AllocateRaw`：
```c++
if (allocation.To(&object)) {
    ...
    OnAllocationEvent(object, size_in_bytes);
  }

  return allocation;
```
这将使我们返回到 `AllocateRawWithLightRetry`：
```c++
AllocationResult alloc = AllocateRaw(size, space, alignment);
if (alloc.To(&result)) {
  DCHECK(result != exception());
  return result;
}
```
这将使我们返回到 `AllocateRawWithRetryOrFail`：
```c++
  HeapObject* result = AllocateRawWithLigthRetry(size, space, alignment);
  if (result) return result;
```
该返回将返回到 `src/heap/factory.cc` 中的 `NewMap`：
```c++
  result->set_map_after_allocation(*meta_map(), SKIP_WRITE_BARRIER);
  return handle(InitializeMap(Map::cast(result), type, instance_size,
                              elements_kind, inobject_properties),
                isolate());
```
`InitializeMap`：
```c++
  map->set_instance_type(type);
```
接下来在 `Map::CopyWithField` 中：
```c++
  Handle<Map> new_map = Map::CopyAddDescriptor(map, &d, flag);
```
`CopyAddDescriptor` 执行：
```c++
  Handle<DescriptorArray> descriptors(map->instance_descriptors());
 
  int nof = map->NumberOfOwnDescriptors();
  Handle<DescriptorArray> new_descriptors = DescriptorArray::CopyUpTo(descriptors, nof, 1);
  new_descriptors->Append(descriptor);
  
  Handle<LayoutDescriptor> new_layout_descriptor =
      FLAG_unbox_double_fields
          ? LayoutDescriptor::New(map, new_descriptors, nof + 1)
          : handle(LayoutDescriptor::FastPointerLayout(), map->GetIsolate());

  return CopyReplaceDescriptors(map, new_descriptors, new_layout_descriptor,
                                flag, descriptor->GetKey(), "CopyAddDescriptor",
                                SIMPLE_PROPERTY_TRANSITION);
```
让我们仔细看看 `LayoutDescriptor`：
```console
(lldb) expr new_layout_descriptor->Print()
Layout descriptor: <all tagged>
```
TODO：仔细查看 LayoutDescriptor。

稍后在 `Object::AddDataProperty` 中实际添加值时：
```c++
  it->WriteDataValue(value, true);
```
此调用最终将进入 `src/lookup.cc`，在我们的例子中，路径将是以下调用：
```c++
  JSObject::cast(*holder)->WriteToField(descriptor_number(), property_details_, *value);
```
TODO：仔细查看 LookupIterator。
`WriteToField` 可以在 `src/objects-inl.h` 中找到：
```c++
  FieldIndex index = FieldIndex::ForDescriptor(map(), descriptor);
```
`FieldIndex::ForDescriptor` 可以在 `src/field-index-inl.h` 中找到：
```c++
inline FieldIndex FieldIndex::ForDescriptor(const Map* map, int descriptor_index) {
  PropertyDetails details = map->instance_descriptors()->GetDetails(descriptor_index);
  int field_index = details.field_index();
  return ForPropertyIndex(map, field_index, details.representation());
}
```
注意这是在传入的 map 上调用 `instance_descriptors()`。正如我们之前回想的，这返回一个 DescriptorArray（它是 WeakFixedArray 的一种类型）。

我们的 DescriptorArray 只有一个条目：
```console
(lldb) expr map->instance_descriptors()->number_of_descriptors()
(int) $6 = 1
(lldb) expr map->instance_descriptors()->GetKey(0)->Print()
#prop_name
(lldb) expr map->instance_descriptors()->GetFieldIndex(0)
(int) $11 = 0
```
我们也可以在 DescriptorArray 上使用 `Print`：
```console
lldb) expr map->instance_descriptors()->Print()

  [0]: #prop_name (data field 0:h, p: 0, attrs: [WEC]) @ Any
```
在我们的例子中，我们正在访问 PropertyDetails，然后获取 `field_index`，我认为它告诉我们该属性的值存储在对象中的什么位置。
`ForDescriptor` 中的最后一个调用是 `ForPropertyIndex`：
```c++
inline FieldIndex FieldIndex::ForPropertyIndex(const Map* map,
                                               int property_index,
                                               Representation representation) {
  int inobject_properties = map->GetInObjectProperties();
  bool is_inobject = property_index < inobject_properties;
  int first_inobject_offset;
  int offset;
  if (is_inobject) {
    first_inobject_offset = map->GetInObjectPropertyOffset(0);
    offset = map->GetInObjectPropertyOffset(property_index);
  } else {
    first_inobject_offset = FixedArray::kHeaderSize;
    property_index -= inobject_properties;
    offset = FixedArray::kHeaderSize + property_index * kPointerSize;
  }
  Encoding encoding = FieldEncoding(representation);
  return FieldIndex(is_inobject, offset, encoding, inobject_properties,
                    first_inobject_offset);
}
```
我原以为这里的 `inobject_properties` 会是 1，但它是 0：
```console
(lldb) expr inobject_properties
(int) $14 = 0
```
为什么会这样，我错过了什么？
这些对象内属性（In-object properties）直接存储在对象实例上，不使用属性数组。稍后我将通过一个例子来澄清这一点。
TODO：添加对象内属性示例。

回到 `JSObject::WriteToField`：
```c++
  RawFastPropertyAtPut(index, value);
```

```c++
void JSObject::RawFastPropertyAtPut(FieldIndex index, Object* value) {
  if (index.is_inobject()) {
    int offset = index.offset();
    WRITE_FIELD(this, offset, value);
    WRITE_BARRIER(GetHeap(), this, offset, value);
  } else {
    property_array()->set(index.outobject_array_index(), value);
  }
}
```
在我们的例子中，我们知道 index 不是 `inobject()`：
```console
(lldb) expr index.is_inobject()
(bool) $18 = false
```
因此，将调用 `property_array()->set()`。
```console
(lldb) expr this
(v8::internal::JSObject *) $21 = 0x00002c31c6a88b59
```
JSObject 继承自 JSReceiver，后者是声明 `property_array()` 函数的地方。
```c++
  inline PropertyArray* property_array() const;
```
```console
(lldb) expr property_array()->Print()
0x2c31c6a88bb1: [PropertyArray]
 - map: 0x2c31f5603e21 <Map>
 - length: 3
 - hash: 0
           0: 0x2c31f56025a1 <Odd Oddball: uninitialized>
         1-2: 0x2c31f56026f1 <undefined>
(lldb) expr index.outobject_array_index()
(int) $26 = 0
(lldb) expr value->Print()
#prop_value
```
观察打印出的值，我们应该看到属性被写入到第 0 个条目。
```console
(lldb) expr property_array()->get(0)->Print()
#uninitialized
// 调用 set 之后
(lldb) expr property_array()->get(0)->Print()
#prop_value
```

```console
(lldb) expr map->instance_descriptors()
(v8::internal::DescriptorArray *) $4 = 0x000039a927082339
```
因此，Map 有一个指向 DescriptorArray 实例的指针数组。

```console
(lldb) expr map->GetInObjectProperties()
(int) $19 = 1
```
每个 Map 都有一个整数，告诉我们它拥有的属性数量。这是创建新 Map 时指定的值，例如：
```console
i::Handle<i::Map> map = i::Map::Create(asInternal(isolate_), 1);
```
但在现阶段，我们实际上还没有任何属性。属性的值与 Object 的实际实例相关联。Map 指定的是特定属性值的索引。

#### 创建 Map 实例
让我们看看 Map 是何时创建的。
```console
(lldb) br s -f map_test.cc -l 63
```

```c++
Handle<Map> Factory::NewMap(InstanceType type,
                            int instance_size,
                            ElementsKind elements_kind,
                            int inobject_properties) {
  HeapObject* result = isolate()->heap()->AllocateRawWithRetryOrFail(Map::kSize, MAP_SPACE);
  result->set_map_after_allocation(*meta_map(), SKIP_WRITE_BARRIER);
  return handle(InitializeMap(Map::cast(result), type, instance_size,
                              elements_kind, inobject_properties),
                isolate());
}
```
我们可以看到上面调用了堆实例上的 `AllocateRawWithRetryOrFail`，传入大小 `88` 并指定 `MAP_SPACE`：
```c++
HeapObject* Heap::AllocateRawWithRetryOrFail(int size, AllocationSpace space,
                                             AllocationAlignment alignment) {
  AllocationResult alloc;
  HeapObject* result = AllocateRawWithLigthRetry(size, space, alignment);
  if (result) return result;

  isolate()->counters()->gc_last_resort_from_handles()->Increment();
  CollectAllAvailableGarbage(GarbageCollectionReason::kLastResort);
  {
    AlwaysAllocateScope scope(isolate());
    alloc = AllocateRaw(size, space, alignment);
  }
  if (alloc.To(&result)) {
    DCHECK(result != exception());
    return result;
  }
  // TODO(1181417): Fix this.
  FatalProcessOutOfMemory("CALL_AND_RETRY_LAST");
  return nullptr;
}
```
`alignment` 的默认值是 `kWordAligned`。阅读头文件中的文档，它说该函数将尝试在 `MAP_SPACE` 中执行大小为 `88` 的分配，如果失败，将执行完整的 GC 并重试分配。
让我们看看 `AllocateRawWithLigthRetry`：
```c++
  AllocationResult alloc = AllocateRaw(size, space, alignment);
```
`AllocateRaw` 可以在 `src/heap/heap-inl.h` 中找到。根据 `space` 参数，会采取不同的路径。由于在我们的例子中是 `MAP_SPACE`，我们将关注该路径：
```c++
AllocationResult Heap::AllocateRaw(int size_in_bytes, AllocationSpace space, AllocationAlignment alignment) {
  ...
  HeapObject* object = nullptr;
  AllocationResult allocation;
  if (OLD_SPACE == space) {
  ...
  } else if (MAP_SPACE == space) {
    allocation = map_space_->AllocateRawUnaligned(size_in_bytes);
  }
  ...
}
```
`map_space_` 是 Heap 的私有成员（`src/heap/heap.h`）：
```c++
MapSpace* map_space_;
```
`AllocateRawUnaligned` 可以在 `src/heap/spaces-inl.h` 中找到：
```c++
AllocationResult PagedSpace::AllocateRawUnaligned( int size_in_bytes, UpdateSkipList update_skip_list) {
  if (!EnsureLinearAllocationArea(size_in_bytes)) {
    return AllocationResult::Retry(identity());
  }

  HeapObject* object = AllocateLinearly(size_in_bytes);
  MSAN_ALLOCATED_UNINITIALIZED_MEMORY(object->address(), size_in_bytes);
  return object;
}
```
`update_skip_list` 的默认值是 `UPDATE_SKIP_LIST`。
让我们看看 `AllocateLinearly`：
```c++
HeapObject* PagedSpace::AllocateLinearly(int size_in_bytes) {
  Address current_top = allocation_info_.top();
  Address new_top = current_top + size_in_bytes;
  allocation_info_.set_top(new_top);
  return HeapObject::FromAddress(current_top);
}
```
回想一下，在我们的例子中 `size_in_bytes` 是 `88`。
```console
(lldb) expr current_top
(v8::internal::Address) $5 = 24847457492680
(lldb) expr new_top
(v8::internal::Address) $6 = 24847457492768
(lldb) expr new_top - current_top
(unsigned long) $7 = 88
```
注意，首先将 top 设置为 new_top，然后返回 current_top，这将是一个指向内存中对象起始位置的指针（在这种情况下是 `v8::internal::Map`，它也是 HeapObject 类型）。
我一直在想为什么 Map（和其他 HeapObject）没有任何成员字段，而只有/主要是各种组成对象的字段的 getter/setter。答案是，指向例如 Map 实例的指针指向实例的第一个内存位置。而 getter/setter 函数使用索引来读写内存位置。索引主要以枚举字段的形式存在，定义了类型的内存布局。

接下来，在 `AllocateRawUnaligned` 中我们有 `MSAN_ALLOCATED_UNINITIALIZED_MEMORY` 宏：
```c++
  MSAN_ALLOCATED_UNINITIALIZED_MEMORY(object->address(), size_in_bytes);
```
`MSAN_ALLOCATED_UNINITIALIZED_MEMORY` 可以在 `src/msan.h` 中找到，`ms` 代表 `Memory Sanitizer`，仅在定义了 `V8_US_MEMORY_SANITIZER` 时使用。
返回的 `object` 在返回时将用于构造 `AllocationResult`。
回到 `AllocateRaw`：
```c++
if (allocation.To(&object)) {
    ...
    OnAllocationEvent(object, size_in_bytes);
  }

  return allocation;
```
这将使我们返回到 `AllocateRawWithLightRetry`：
```c++
AllocationResult alloc = AllocateRaw(size, space, alignment);
if (alloc.To(&result)) {
  DCHECK(result != exception());
  return result;
}
```
这将使我们返回到 `AllocateRawWithRetryOrFail`：
```c++
  HeapObject* result = AllocateRawWithLigthRetry(size, space, alignment);
  if (result) return result;
```
该返回将返回到 `src/heap/factory.cc` 中的 `NewMap`：
```c++
  result->set_map_after_allocation(*meta_map(), SKIP_WRITE_BARRIER);
  return handle(InitializeMap(Map::cast(result), type, instance_size,
                              elements_kind, inobject_properties),
                isolate());
```
`InitializeMap`：
```c++
  map->set_instance_type(type);
  map->set_prototype(*null_value(), SKIP_WRITE_BARRIER);
  map->set_constructor_or_backpointer(*null_value(), SKIP_WRITE_BARRIER);
  map->set_instance_size(instance_size);
  if (map->IsJSObjectMap()) {
    DCHECK(!isolate()->heap()->InReadOnlySpace(map));
    map->SetInObjectPropertiesStartInWords(instance_size / kPointerSize - inobject_properties);
    DCHECK_EQ(map->GetInObjectProperties(), inobject_properties);
    map->set_prototype_validity_cell(*invalid_prototype_validity_cell());
  } else {
    DCHECK_EQ(inobject_properties, 0);
    map->set_inobject_properties_start_or_constructor_function_index(0);
    map->set_prototype_validity_cell(Smi::FromInt(Map::kPrototypeChainValid));
  }
  map->set_dependent_code(DependentCode::cast(*empty_fixed_array()), SKIP_WRITE_BARRIER);
  map->set_weak_cell_cache(Smi::kZero);
  map->set_raw_transitions(MaybeObject::FromSmi(Smi::kZero));
  map->SetInObjectUnusedPropertyFields(inobject_properties);
  map->set_instance_descriptors(*empty_descriptor_array());

  map->set_visitor_id(Map::GetVisitorId(map));
  map->set_bit_field(0);
  map->set_bit_field2(Map::IsExtensibleBit::kMask);
  int bit_field3 = Map::EnumLengthBits::encode(kInvalidEnumCacheSentinel) |
                   Map::OwnsDescriptorsBit::encode(true) |
                   Map::ConstructionCounterBits::encode(Map::kNoSlackTracking);
  map->set_bit_field3(bit_field3);
  map->set_elements_kind(elements_kind); //HOLEY_ELEMENTS
  map->set_new_target_is_base(true);
  isolate()->counters()->maps_created()->Increment();
  if (FLAG_trace_maps) LOG(isolate(), MapCreate(map));
  return map;
```

### Context
Context 继承自 `FixedArray`（`src/context.h`）。因此，Context 的实例是一个 FixedArray，我们可以使用 `Get(index)` 等方法来获取数组中的条目。

### V8_EXPORT
这在 V8 源代码的许多地方都可以看到。例如：
```c++
    class V8_EXPORT ArrayBuffer : public Object {
```
这是什么？
它是一个预处理器宏，看起来像这样：
```c++
    #if V8_HAS_ATTRIBUTE_VISIBILITY && defined(V8_SHARED)
    # ifdef BUILDING_V8_SHARED
    #  define V8_EXPORT __attribute__ ((visibility("default")))
    # else
    #  define V8_EXPORT
    # endif
    #else
    # define V8_EXPORT
    #endif 
```
我们可以看到，如果定义了 `V8_HAS_ATTRIBUTE_VISIBILITY` 和 `V8_SHARED`，并且如果定义了 `BUILDING_V8_SHARED`，`V8_EXPORT` 就被设置为 `__attribute__ ((visibility("default")))`。但在所有其他情况下，`V8_EXPORT` 为空，预处理器不会插入任何内容。
那么 `__attribute__ ((visibility("default")))` 是什么？

在 GNU 编译器集合（GCC）环境中，用于导出的术语是可见性（Visibility）。当它应用于共享对象中的函数和变量时，可见性是指其他共享对象调用 C/C++ 函数的能力。具有默认可见性（Default visibility）的函数具有全局作用域，可以从其他共享对象中调用。具有隐藏可见性（Hidden visibility）的函数具有局部作用域，不能从其他共享对象中调用。

可见性可以通过使用编译器选项或可见性属性来控制。
这里我们可以看到我们的 JavaScript 作为一个 String。
```c++
    std::unique_ptr<Utf16CharacterStream> stream(ScannerStream::For(source));
    scanner_.Initialize(stream.get(), info->is_module());
    result = DoParseProgram(info);
```
`DoParseProgram`：
```console
    (lldb) br s -f parser.cc -l 639
    ...

    this->scope()->SetLanguageMode(info->language_mode());
    ParseStatementList(body, Token::EOS, &ok);
```
此调用将进入 `parser-base.h` 及其 `ParseStatementList` 函数。
```console
    (lldb) br s -f parser-base.h -l 4695

    StatementT stat = ParseStatementListItem(CHECK_OK_CUSTOM(Return, kLazyParsingComplete));

    result = CompileToplevel(&parse_info, isolate, Handle<SharedFunctionInfo>::null());
```
这将进入 `CompileToplevel`（在同一个文件 `src/compiler.cc` 中）：
```c++
    // 编译代码。
    result = CompileUnoptimizedCode(parse_info, shared_info, isolate);
```
这将进入 `CompileUnoptimizedCode`（在同一个文件 `src/compiler.cc` 中）：
```c++
    // 准备并执行最外层函数的非优化编译任务。
    std::unique_ptr<CompilationJob> outer_job(
       PrepareAndExecuteUnoptimizedCompileJob(parse_info, parse_info->literal(),
                                              shared_info, isolate));


    std::unique_ptr<CompilationJob> job(
        interpreter::Interpreter::NewCompilationJob(parse_info, literal, isolate));
    if (job->PrepareJob() == CompilationJob::SUCCEEDED &&
        job->ExecuteJob() == CompilationJob::SUCCEEDED) {
      return job;
    }
```
`PrepareJobImpl`：
```c++
    CodeGenerator::MakeCodePrologue(parse_info(), compilation_info(),
                                    "interpreter");
    return SUCCEEDED;
```
`codegen.cc` 中的 `MakeCodePrologue`：

`interpreter.cc` 中的 `ExecuteJobImpl`：
```c++
    generator()->GenerateBytecode(stack_limit());    
```
`src/interpreter/bytecode-generator.cc`：
```c++
     RegisterAllocationScope register_scope(this);
```
字节码是基于寄存器的（如果这个术语正确的话），我们之前有一个例子。我猜这就是这个调用的目的。

`VisitDeclarations` 将遍历文件中的所有声明，在我们的例子中是：
```c++
    var user1 = new Person('Fletch');
    var user2 = new Person('Dr.Rosen');

    (lldb) p *variable->raw_name()
    (const v8::internal::AstRawString) $33 = {
       = {
        next_ = 0x000000010600a280
        string_ = 0x000000010600a280
      }
      literal_bytes_ = (start_ = "user1", length_ = 5)
      hash_field_ = 1303438034
      is_one_byte_ = true
      has_string_ = false
    }

    // 在主体之前执行栈检查。
    builder()->StackCheck(info()->literal()->start_position());
```
因此，该调用将输出一个 `StackCheck` 指令，如上面的示例所示：
```
    14 E> 0x2eef8d9b103e @    0 : 7f                StackCheck
```

### 性能
假设你有一个表达式 `x + y`，`full-codegen` 编译器可能会产生：
```
    movq rax, x
    movq rbx, y
    callq RuntimeAdd
```
如果 `x` 和 `y` 是整数，直接使用 `add` 操作会快得多：
```
    movq rax, x
    movq rbx, y
    add rax, rbx
```
回想一下，函数是经过优化的，因此如果编译器不得不放弃优化（bail out）并对函数的一部分进行去优化（unoptimize），那么整个函数都会受到影响，并回到非优化版本。

## Bytecode
本节将检查以下 JavaScript 的字节码：
```js
    function beve() {
      const p = new Promise((resolve, reject) => {
        resolve('ok');
      });

      p.then(msg => {
        console.log(msg);
      });
    }

    beve(); 
```
```shell
    $ d8 --print-bytecode promise.js
```
首先是主函数，它没有名字：
```
    [generating bytecode for function: ]
    (生成此代码的代码可以在 src/objects.cc 的 BytecodeArray::Dissassemble 中找到)
    Parameter count 1
    Frame size 32
           // 将常量池中的 FixedArray[4] 加载到累加器中。
           0x34423e7ac19e @    0 : 09 00             LdaConstant [0] 
           // 将 FixedArray[4] 存储在寄存器 r1 中。
           0x34423e7ac1a0 @    2 : 1e f9             Star r1
           // 将零加载到累加器中。
           0x34423e7ac1a2 @    4 : 02                LdaZero
           // 将零（累加器的内容）存储到寄存器 r2 中。
           0x34423e7ac1a3 @    5 : 1e f8             Star r2
           // 
           0x34423e7ac1a5 @    7 : 1f fe f7          Mov <closure>, r3
           0x34423e7ac1a8 @   10 : 53 96 01 f9 03    CallRuntime [DeclareGlobalsForInterpreter], r1-r3
      0 E> 0x34423e7ac1ad @   15 : 90                StackCheck
    141 S> 0x34423e7ac1ae @   16 : 0a 01 00          LdaGlobal [1], [0]
           0x34423e7ac1b1 @   19 : 1e f9             Star r1
    141 E> 0x34423e7ac1b3 @   21 : 4f f9 03          CallUndefinedReceiver0 r1, [3]
           0x34423e7ac1b6 @   24 : 1e fa             Star r0
    148 S> 0x34423e7ac1b8 @   26 : 94                Return

    Constant pool (size = 2)
    0x34423e7ac149: [FixedArray] in OldSpace
     - map = 0x344252182309 <Map(HOLEY_ELEMENTS)>
     - length: 2
           0: 0x34423e7ac069 <FixedArray[4]>
           1: 0x34423e7abf59 <String[4]: beve>
```
Handler Table (size = 16) 将常量池条目 `<name_index>` 中的全局变量加载到累加器中，使用 `typeof` 之外的 `FeedBackVector` 插槽 `<slot>`。

* `LdaConstant <idx>` 
从常量池中加载索引处的常量到累加器。
* `Star <dst>`
将累加器寄存器的内容存储在 `dst` 中。
* `Ldar <src>`
将寄存器 `src` 的值加载到累加器。
* `LdaGlobal <idx> <slot>`
将常量池条目 `idx` 中的全局变量加载到累加器中，使用 `typeof` 之外的 `FeedBackVector` 插槽。
* `Mov <closure>, <r3>`
存储寄存器的值。

你可以在 `src/interpreter/interpreter-generator.cc` 中找到这些指令的声明。

## 统一代码生成架构

## FeedbackVector
附加到每个函数，负责记录和管理所有的执行反馈，即关于类型启用的信息。
你可以在 `src/feedback-vector.h` 中找到该类的声明。

## BytecodeGenerator
目前是 V8 中唯一关心 AST 的部分。

## BytecodeGraphBuilder
基于解释器字节码生成高级 IR 图。

## TurboFan
是一个编译器后端，接收控制流图，然后进行指令选择、寄存器分配和代码生成。

### 执行/运行时
我不确定 V8 是否严格遵循这一点，但我听说并读到，当引擎遇到函数声明时，它只解析并验证语法，并保存对函数名的引用。函数内部的语句在此阶段不会被检查，只检查函数声明的语法（括号、参数、大括号等）。

### 函数方法
`Function` 的声明可以在 `include/v8.h` 中找到（仅作记录，因为我找了好几次）。

### Symbol
`Symbol` 类的声明可以在 `v8.h` 中找到，内部实现在 `src/api/api.cc` 中。

众所周知的 Symbol 是使用宏生成的，因此你无法仅通过搜索静态函数名（如 `GetToPrimitive`）来找到它们。
```c++
#define WELL_KNOWN_SYMBOLS(V)                 \
  V(AsyncIterator, async_iterator)            \
  V(HasInstance, has_instance)                \
  V(IsConcatSpreadable, is_concat_spreadable) \
  V(Iterator, iterator)                       \
  V(Match, match)                             \
  V(Replace, replace)                         \
  V(Search, search)                           \
  V(Split, split)                             \
  V(ToPrimitive, to_primitive)                \
  V(ToStringTag, to_string_tag)               \
  V(Unscopables, unscopables)

#define SYMBOL_GETTER(Name, name)                                   \
  Local<Symbol> v8::Symbol::Get##Name(Isolate* isolate) {           \
    i::Isolate* i_isolate = reinterpret_cast<i::Isolate*>(isolate); \
    return Utils::ToLocal(i_isolate->factory()->name##_symbol());   \
  }
```
因此 `GetToPrimitive` 将变为：
```c++
Local<Symbol> v8::Symbol::GeToPrimitive(Isolate* isolate) {
  i::Isolate* i_isolate = reinterpret_cast<i::Isolate*>(isolate);
  return Utils::ToLocal(i_isolate->factory()->to_primitive_symbol());
}
```
在 [symbol-test.cc](./test/symbol-test.cc) 中有一个示例。

## Builtins
是由 V8 提供的 JavaScript 函数/对象。这些是使用 C++ DSL 构建的，并经过：
`CodeStubAssembler -> CodeAssembler -> RawMachineAssembler`。

Builtins 需要为其生成字节码，以便它们可以在 TurboFan 中运行。

`src/code-stub-assembler.h`

所有的 Builtins 都在 `src/builtins/builtins-definitions.h` 中通过 `BUILTIN_LIST_BASE` 宏声明。
有不同类型的 Builtins（TF = Turbo Fan）：
* TFJ 
JavaScript 链接，这意味着它可以作为 JavaScript 函数调用。
* TFS
CodeStub 链接。具有 Stub 链接的 Builtin 可用于将通用代码提取到单独的代码对象中，然后可由多个调用者使用。这很有用，因为 Builtins 是在编译时生成的，并包含在 V8 快照中。这意味着它们是创建的每个 Isolate 的一部分。能够为多个 Builtins 共享通用代码将节省空间。
* TFC 
具有自定义描述符的 CodeStub 链接。

要查看其运行情况，我们首先需要禁用快照。如果不禁用，我们将无法设置断点，因为堆将在编译时序列化，并在 V8 启动时反序列化。

要找到禁用快照的选项，请使用：
```shell
    $ gn args --list out.gn/learning --short | more
    ...
    v8_use_snapshot=true
    $ gn args out.gn/learning
    v8_use_snapshot=false
    $ gn -C out.gn/learning
```
构建后，我们应该能够在 `bootstrapper.cc` 及其函数 `Genesis::InitializeGlobal` 中设置断点：
```console
    (lldb) br s -f bootstrapper.cc -l 2684
```
让我们看看 `JSON` 对象是如何设置的：
```c++
    Handle<String> name = factory->InternalizeUtf8String("JSON");
    Handle<JSObject> json_object = factory->NewJSObject(isolate->object_function(), TENURED);
```
`TENURED` 意味着该对象应直接分配在老生代（Old generation）中。
```c++
    JSObject::AddProperty(global, name, json_object, DONT_ENUM);
```
`DONT_ENUM` 会被一些 Builtin 函数检查，如果设置了，该对象将被这些函数忽略。
```c++
    SimpleInstallFunction(json_object, "parse", Builtins::kJsonParse, 2, false);
```
这里我们可以看到我们正在安装一个名为 `parse` 的函数，它接受 2 个参数。你可以在 `src/builtins/builtins-json.cc` 中找到定义。
`SimpleInstallFunction` 是做什么的？

以 `console` 为例，它是使用以下代码创建的：
```c++
    Handle<JSObject> console = factory->NewJSObject(cons, TENURED);
    JSObject::AddProperty(global, name, console, DONT_ENUM);
    SimpleInstallFunction(console, "debug", Builtins::kConsoleDebug, 1, false,
                          NONE);

    V8_NOINLINE Handle<JSFunction> SimpleInstallFunction(
      Handle<JSObject> base, 
      const char* name, 
      Builtins::Name call, 
      int len,
      bool adapt, 
      PropertyAttributes attrs = DONT_ENUM,
      BuiltinFunctionId id = kInvalidBuiltinFunctionId) {
```
我们可以看到 `base` 是我们指向 JSObject 的 Handle，`name` 是 "debug"。
`Builtins::Name` 是 `Builtins:kConsoleDebug`。这是在哪里定义的？
你可以在 `src/builtins/builtins-definitions.h` 中找到一个名为 `CPP` 的宏：
```c++
   CPP(ConsoleDebug)
```
这个宏展开后是什么？
它是 `builtin-definitions.h` 中 `BUILTIN_LIST_BASE` 宏的一部分。
我们必须查看 `BUILTIN_LIST` 在哪里被使用，我们可以在 `builtins.cc` 中找到。
在 `builtins.cc` 中，我们有一个 `BuiltinMetadata` 数组，声明如下：
```c++
    const BuiltinMetadata builtin_metadata[] = {
      BUILTIN_LIST(DECL_CPP, DECL_API, DECL_TFJ, DECL_TFC, DECL_TFS, DECL_TFH, DECL_ASM)
    };

    #define DECL_CPP(Name, ...) { #Name, Builtins::CPP, \
                                { FUNCTION_ADDR(Builtin_##Name) }},
```
这将展开为在数组中创建一个 `BuiltinMetadata` 结构体条目。`BuiltinMetadata` 结构体如下所示，这可能有助于理解发生了什么：
```c++
    struct BuiltinMetadata {
      const char* name;
      Builtins::Kind kind;
      union {
        Address cpp_entry;       // 用于 CPP 和 API builtins。
        int8_t parameter_count;  // 用于 TFJ builtins。
      } kind_specific_data;
    };
```
因此 `CPP(ConsoleDebug)` 将展开为数组中的一个条目，看起来像这样：
```c++
    { ConsoleDebug, 
      Builtins::CPP, 
      {
        reinterpret_cast<v8::internal::Address>(reinterpret_cast<intptr_t>(Builtin_ConsoleDebug))
      }
    },
```
第三个参数是 union 的创建，这可能并不明显。

回到我试图回答的问题：
“`Builtins::Name` 是 `Builtins:kConsoleDebug`。这是在哪里定义的？”
为此，我们必须查看 `builtins.h` 和枚举 `Name`：
```c++
    enum Name : int32_t {
    #define DEF_ENUM(Name, ...) k##Name,
        BUILTIN_LIST_ALL(DEF_ENUM)
    #undef DEF_ENUM
        builtin_count
     };
```
这将使用 `DEF_ENUM` 宏展开为 `builtin-definitions.h` 中的完整 Builtins 列表。因此 `ConsoleDebug` 的展开将如下所示：
```c++
    enum Name: int32_t {
      ...
      kDebugConsole,
      ...
    };
```
回到 `SimpleInstallFunction` 的参数：
```c++
    SimpleInstallFunction(console, "debug", Builtins::kConsoleDebug, 1, false,
                          NONE);

    V8_NOINLINE Handle<JSFunction> SimpleInstallFunction(
      Handle<JSObject> base, 
      const char* name, 
      Builtins::Name call, 
      int len,
      bool adapt, 
      PropertyAttributes attrs = DONT_ENUM,
      BuiltinFunctionId id = kInvalidBuiltinFunctionId) {
```
我们了解了 `Builtins::Name`，那么看看 `len`（为 1），这是什么？
`SimpleInstallFunction` 将调用：
```c++
    Handle<JSFunction> fun =
      SimpleCreateFunction(base->GetIsolate(), function_name, call, len, adapt);
```
如果 `adapt` 为 true，则会使用 `len`，但在我们的例子中它为 false。如果 `adapt` 为 true，它将用于：
```c++
    fun->shared()->set_internal_formal_parameter_count(len);
```
我不确定这里的 `adapt` 指的是什么。

`PropertyAttributes` 未指定，因此它将获得默认值 `DONT_ENUM`。
最后一个参数类型为 `BuiltinFunctionId`，也未指定，因此将使用默认值 `kInvalidBuiltinFunctionId`。这是一个定义在 `src/objects/objects.h` 中的枚举。

这篇 [博客](https://v8project.blogspot.se/2017/11/csa.html) 讨论了 CSA。
在这里我们可以看到我们的 JavaScript 作为一个 String。
```c++
    std::unique_ptr<Utf16CharacterStream> stream(ScannerStream::For(source));
    scanner_.Initialize(stream.get(), info->is_module());
    result = DoParseProgram(info);
```
`DoParseProgram`：
```console
    (lldb) br s -f parser.cc -l 639
    ...

    this->scope()->SetLanguageMode(info->language_mode());
    ParseStatementList(body, Token::EOS, &ok);
```
此调用将进入 `parser-base.h` 及其 `ParseStatementList` 函数。
```console
    (lldb) br s -f parser-base.h -l 4695

    StatementT stat = ParseStatementListItem(CHECK_OK_CUSTOM(Return, kLazyParsingComplete));

    result = CompileToplevel(&parse_info, isolate, Handle<SharedFunctionInfo>::null());
```
这将进入 `CompileToplevel`（在同一个文件 `src/compiler.cc` 中）：
```c++
    // 编译代码。
    result = CompileUnoptimizedCode(parse_info, shared_info, isolate);
```
这将进入 `CompileUnoptimizedCode`（在同一个文件 `src/compiler.cc` 中）：
```c++
    // 准备并执行最外层函数的非优化编译任务。
    std::unique_ptr<CompilationJob> outer_job(
       PrepareAndExecuteUnoptimizedCompileJob(parse_info, parse_info->literal(),
                                              shared_info, isolate));


    std::unique_ptr<CompilationJob> job(
        interpreter::Interpreter::NewCompilationJob(parse_info, literal, isolate));
    if (job->PrepareJob() == CompilationJob::SUCCEEDED &&
        job->ExecuteJob() == CompilationJob::SUCCEEDED) {
      return job;
    }
```
`PrepareJobImpl`：
```c++
    CodeGenerator::MakeCodePrologue(parse_info(), compilation_info(),
                                    "interpreter");
    return SUCCEEDED;
```
`codegen.cc` 中的 `MakeCodePrologue`：

`interpreter.cc` 中的 `ExecuteJobImpl`：
```c++
    generator()->GenerateBytecode(stack_limit());    
```
`src/interpreter/bytecode-generator.cc`：
```c++
     RegisterAllocationScope register_scope(this);
```
字节码是基于寄存器的（如果这个术语正确的话），我们之前有一个例子。我猜这就是这个调用的目的。

`VisitDeclarations` 将遍历文件中的所有声明，在我们的例子中是：
```c++
    var user1 = new Person('Fletch');
    var user2 = new Person('Dr.Rosen');

    (lldb) p *variable->raw_name()
    (const v8::internal::AstRawString) $33 = {
       = {
        next_ = 0x000000010600a280
        string_ = 0x000000010600a280
      }
      literal_bytes_ = (start_ = "user1", length_ = 5)
      hash_field_ = 1303438034
      is_one_byte_ = true
      has_string_ = false
    }

    // 在主体之前执行栈检查。
    builder()->StackCheck(info()->literal()->start_position());
```
因此，该调用将输出一个 `StackCheck` 指令，如上面的示例所示：
```
    14 E> 0x2eef8d9b103e @    0 : 7f                StackCheck
```

### 性能
假设你有一个表达式 `x + y`，`full-codegen` 编译器可能会产生：
```
    movq rax, x
    movq rbx, y
    callq RuntimeAdd
```
如果 `x` 和 `y` 是整数，直接使用 `add` 操作会快得多：
```
    movq rax, x
    movq rbx, y
    add rax, rbx
```
回想一下，函数是经过优化的，因此如果编译器不得不放弃优化（bail out）并对函数的一部分进行去优化（unoptimize），那么整个函数都会受到影响，并回到非优化版本。

## Bytecode
本节将检查以下 JavaScript 的字节码：
```js
    function beve() {
      const p = new Promise((resolve, reject) => {
        resolve('ok');
      });

      p.then(msg => {
        console.log(msg);
      });
    }

    beve(); 
```
```shell
    $ d8 --print-bytecode promise.js
```
首先是主函数，它没有名字：
```
    [generating bytecode for function: ]
    (生成此代码的代码可以在 src/objects.cc 的 BytecodeArray::Dissassemble 中找到)
    Parameter count 1
    Frame size 32
           // 将常量池中的 FixedArray[4] 加载到累加器中。
           0x34423e7ac19e @    0 : 09 00             LdaConstant [0] 
           // 将 FixedArray[4] 存储在寄存器 r1 中。
           0x34423e7ac1a0 @    2 : 1e f9             Star r1
           // 将零加载到累加器中。
           0x34423e7ac1a2 @    4 : 02                LdaZero
           // 将零（累加器的内容）存储到寄存器 r2 中。
           0x34423e7ac1a3 @    5 : 1e f8             Star r2
           // 
           0x34423e7ac1a5 @    7 : 1f fe f7          Mov <closure>, r3
           0x34423e7ac1a8 @   10 : 53 96 01 f9 03    CallRuntime [DeclareGlobalsForInterpreter], r1-r3
      0 E> 0x34423e7ac1ad @   15 : 90                StackCheck
    141 S> 0x34423e7ac1ae @   16 : 0a 01 00          LdaGlobal [1], [0]
           0x34423e7ac1b1 @   19 : 1e f9             Star r1
    141 E> 0x34423e7ac1b3 @   21 : 4f f9 03          CallUndefinedReceiver0 r1, [3]
           0x34423e7ac1b6 @   24 : 1e fa             Star r0
    148 S> 0x34423e7ac1b8 @   26 : 94                Return

    Constant pool (size = 2)
    0x34423e7ac149: [FixedArray] in OldSpace
     - map = 0x344252182309 <Map(HOLEY_ELEMENTS)>
     - length: 2
           0: 0x34423e7ac069 <FixedArray[4]>
           1: 0x34423e7abf59 <String[4]: beve>
```
Handler Table (size = 16) 将常量池条目 `<name_index>` 中的全局变量加载到累加器中，使用 `typeof` 之外的 `FeedBackVector` 插槽 `<slot>`。

* `LdaConstant <idx>` 
从常量池中加载索引处的常量到累加器。
* `Star <dst>`
将累加器寄存器的内容存储在 `dst` 中。
* `Ldar <src>`
将寄存器 `src` 的值加载到累加器。
* `LdaGlobal <idx> <slot>`
将常量池条目 `idx` 中的全局变量加载到累加器中，使用 `typeof` 之外的 `FeedBackVector` 插槽。
* `Mov <closure>, <r3>`
存储寄存器的值。

你可以在 `src/interpreter/interpreter-generator.cc` 中找到这些指令的声明。

## 统一代码生成架构

## FeedbackVector
附加到每个函数，负责记录和管理所有的执行反馈，即关于类型启用的信息。
你可以在 `src/feedback-vector.h` 中找到该类的声明。

## BytecodeGenerator
目前是 V8 中唯一关心 AST 的部分。

## BytecodeGraphBuilder
基于解释器字节码生成高级 IR 图。

## TurboFan
是一个编译器后端，接收控制流图，然后进行指令选择、寄存器分配和代码生成。

### 执行/运行时
我不确定 V8 是否严格遵循这一点，但我听说并读到，当引擎遇到函数声明时，它只解析并验证语法，并保存对函数名的引用。函数内部的语句在此阶段不会被检查，只检查函数声明的语法（括号、参数、大括号等）。

### 函数方法
`Function` 的声明可以在 `include/v8.h` 中找到（仅作记录，因为我找了好几次）。

### Symbol
`Symbol` 类的声明可以在 `v8.h` 中找到，内部实现在 `src/api/api.cc` 中。

众所周知的 Symbol 是使用宏生成的，因此你无法仅通过搜索静态函数名（如 `GetToPrimitive`）来找到它们。
```c++
#define WELL_KNOWN_SYMBOLS(V)                 \
  V(AsyncIterator, async_iterator)            \
  V(HasInstance, has_instance)                \
  V(IsConcatSpreadable, is_concat_spreadable) \
  V(Iterator, iterator)                       \
  V(Match, match)                             \
  V(Replace, replace)                         \
  V(Search, search)                           \
  V(Split, split)                             \
  V(ToPrimitive, to_primitive)                \
  V(ToStringTag, to_string_tag)               \
  V(Unscopables, unscopables)

#define SYMBOL_GETTER(Name, name)                                   \
  Local<Symbol> v8::Symbol::Get##Name(Isolate* isolate) {           \
    i::Isolate* i_isolate = reinterpret_cast<i::Isolate*>(isolate); \
    return Utils::ToLocal(i_isolate->factory()->name##_symbol());   \
  }
```
因此 `GetToPrimitive` 将变为：
```c++
Local<Symbol> v8::Symbol::GeToPrimitive(Isolate* isolate) {
  i::Isolate* i_isolate = reinterpret_cast<i::Isolate*>(isolate);
  return Utils::ToLocal(i_isolate->factory()->to_primitive_symbol());
}
```
在 [symbol-test.cc](./test/symbol-test.cc) 中有一个示例。

## Builtins
是由 V8 提供的 JavaScript 函数/对象。这些是使用 C++ DSL 构建的，并经过：
`CodeStubAssembler -> CodeAssembler -> RawMachineAssembler`。

Builtins 需要为其生成字节码，以便它们可以在 TurboFan 中运行。

`src/code-stub-assembler.h`

所有的 Builtins 都在 `src/builtins/builtins-definitions.h` 中通过 `BUILTIN_LIST_BASE` 宏声明。
有不同类型的 Builtins（TF = Turbo Fan）：
* TFJ 
JavaScript 链接，这意味着它可以作为 JavaScript 函数调用。
* TFS
CodeStub 链接。具有 Stub 链接的 Builtin 可用于将通用代码提取到单独的代码对象中，然后可由多个调用者使用。这很有用，因为 Builtins 是在编译时生成的，并包含在 V8 快照中。这意味着它们是创建的每个 Isolate 的一部分。能够为多个 Builtins 共享通用代码将节省空间。
* TFC 
具有自定义描述符的 CodeStub 链接。

要查看其运行情况，我们首先需要禁用快照。如果不禁用，我们将无法设置断点，因为堆将在编译时序列化，并在 V8 启动时反序列化。

要找到禁用快照的选项，请使用：
```shell
    $ gn args --list out.gn/learning --short | more
    ...
    v8_use_snapshot=true
    $ gn args out.gn/learning
    v8_use_snapshot=false
    $ gn -C out.gn/learning
```
构建后，我们应该能够在 `bootstrapper.cc` 及其函数 `Genesis::InitializeGlobal` 中设置断点：
```console
    (lldb) br s -f bootstrapper.cc -l 2684
```
让我们看看 `JSON` 对象是如何设置的：
```c++
    Handle<String> name = factory->InternalizeUtf8String("JSON");
    Handle<JSObject> json_object = factory->NewJSObject(isolate->object_function(), TENURED);
```
`TENURED` 意味着该对象应直接分配在老生代（Old generation）中。
```c++
    JSObject::AddProperty(global, name, json_object, DONT_ENUM);
```
`DONT_ENUM` 会被一些 Builtin 函数检查，如果设置了，该对象将被这些函数忽略。
```c++
    SimpleInstallFunction(json_object, "parse", Builtins::kJsonParse, 2, false);
```
这里我们可以看到我们正在安装一个名为 `parse` 的函数，它接受 2 个参数。你可以在 `src/builtins/builtins-json.cc` 中找到定义。
`SimpleInstallFunction` 是做什么的？

以 `console` 为例，它是使用以下代码创建的：
```c++
    Handle<JSObject> console = factory->NewJSObject(cons, TENURED);
    JSObject::AddProperty(global, name, console, DONT_ENUM);
    SimpleInstallFunction(console, "debug", Builtins::kConsoleDebug, 1, false,
                          NONE);

    V8_NOINLINE Handle<JSFunction> SimpleInstallFunction(
      Handle<JSObject> base, 
      const char* name, 
      Builtins::Name call, 
      int len,
      bool adapt, 
      PropertyAttributes attrs = DONT_ENUM,
      BuiltinFunctionId id = kInvalidBuiltinFunctionId) {
```
我们可以看到 `base` 是我们指向 JSObject 的 Handle，`name` 是 "debug"。
`Builtins::Name` 是 `Builtins:kConsoleDebug`。这是在哪里定义的？
你可以在 `src/builtins/builtins-definitions.h` 中找到一个名为 `CPP` 的宏：
```c++
   CPP(ConsoleDebug)
```
这个宏展开后是什么？
它是 `builtin-definitions.h` 中 `BUILTIN_LIST_BASE` 宏的一部分。
我们必须查看 `BUILTIN_LIST` 在哪里被使用，我们可以在 `builtins.cc` 中找到。
在 `builtins.cc` 中，我们有一个 `BuiltinMetadata` 数组，声明如下：
```c++
    const BuiltinMetadata builtin_metadata[] = {
      BUILTIN_LIST(DECL_CPP, DECL_API, DECL_TFJ, DECL_TFC, DECL_TFS, DECL_TFH, DECL_ASM)
    };

    #define DECL_CPP(Name, ...) { #Name, Builtins::CPP, \
                                { FUNCTION_ADDR(Builtin_##Name) }},
```
这将展开为在数组中创建一个 `BuiltinMetadata` 结构体条目。`BuiltinMetadata` 结构体如下所示，这可能有助于理解发生了什么：
```c++
    struct BuiltinMetadata {
      const char* name;
      Builtins::Kind kind;
      union {
        Address cpp_entry;       // 用于 CPP 和 API builtins。
        int8_t parameter_count;  // 用于 TFJ builtins。
      } kind_specific_data;
    };
```
因此 `CPP(ConsoleDebug)` 将展开为数组中的一个条目，看起来像这样：
```c++
    { ConsoleDebug, 
      Builtins::CPP, 
      {
        reinterpret_cast<v8::internal::Address>(reinterpret_cast<intptr_t>(Builtin_ConsoleDebug))
      }
    },
```
第三个参数是 union 的创建，这可能并不明显。

回到我试图回答的问题：
“`Builtins::Name` 是 `Builtins:kConsoleDebug`。这是在哪里定义的？”
为此，我们必须查看 `builtins.h` 和枚举 `Name`：
```c++
    enum Name : int32_t {
    #define DEF_ENUM(Name, ...) k##Name,
        BUILTIN_LIST_ALL(DEF_ENUM)
    #undef DEF_ENUM
        builtin_count
     };
```
这将使用 `DEF_ENUM` 宏展开为 `builtin-definitions.h` 中的完整 Builtins 列表。因此 `ConsoleDebug` 的展开将如下所示：
```c++
    enum Name: int32_t {
      ...
      kDebugConsole,
      ...
    };
```
回到 `SimpleInstallFunction` 的参数：
```c++
    SimpleInstallFunction(console, "debug", Builtins::kConsoleDebug, 1, false,
                          NONE);

    V8_NOINLINE Handle<JSFunction> SimpleInstallFunction(
      Handle<JSObject> base, 
      const char* name, 
      Builtins::Name call, 
      int len,
      bool adapt, 
      PropertyAttributes attrs = DONT_ENUM,
      BuiltinFunctionId id = kInvalidBuiltinFunctionId) {
```
我们了解了 `Builtins::Name`，那么看看 `len`（为 1），这是什么？
`SimpleInstallFunction` 将调用：
```c++
    Handle<JSFunction> fun =
      SimpleCreateFunction(base->GetIsolate(), function_name, call, len, adapt);
```
如果 `adapt` 为 true，则会使用 `len`，但在我们的例子中它为 false。如果 `adapt` 为 true，它将用于：
```c++
    fun->shared()->set_internal_formal_parameter_count(len);
```
我不确定这里的 `adapt` 指的是什么。

`PropertyAttributes` 未指定，因此它将获得默认值 `DONT_ENUM`。
最后一个参数类型为 `BuiltinFunctionId`，也未指定，因此将使用默认值 `kInvalidBuiltinFunctionId`。这是一个定义在 `src/objects/objects.h` 中的枚举。

这篇 [博客](https://v8project.blogspot.se/2017/11/csa.html) 讨论了 CSA。
随后检查参数是否为 `--isolate`、`--module` 或 `-e`，如果不是（如我们的情况）：
```c++
    } else if (strncmp(str, "-", 1) != 0) {
      // 不是标志，因此它必须是要执行的脚本。
      options.script_executed = true;
```
TODO：我不确定 `SourceGroups` 是关于什么的，但先记录下来，稍后会重新审视。

这将带我们回到 `src/d8.cc` 中的 `int Shell::Main`：
```c++
    ::V8::InitializeICUDefaultLocation(argv[0], options.icu_data_file);

    (lldb) p argv[0]
    (char *) $8 = 0x00007fff5fbfed48 "./d8"
```
参见 [ICU](international-component-for-unicode) 了解更多细节。

接下来初始化默认的 V8 平台：
```c++
    g_platform = i::FLAG_verify_predictable ? new PredictablePlatform() : v8::platform::CreateDefaultPlatform();
```
在我们的例子中将调用 `v8::platform::CreateDefaultPlatform()`。

然后我们回到 `Main`，有以下几行：
```c++
    2685 v8::V8::InitializePlatform(g_platform);
    2686 v8::V8::Initialize();
```
这与我在 [Node.js 启动过程](https://github.com/danbev/learning-nodejs#startint-argc-char-argv) 中看到的非常相似。

我们没有在命令行中指定任何 `natives_blob` 或 `snapshot_blob` 作为选项，因此将使用默认值：
```c++
    v8::V8::InitializeExternalStartupData(argv[0]);
```
回到 `src/d8.cc` 第 2918 行：
```c++
    Isolate* isolate = Isolate::New(create_params);
```
此调用将带我们进入 `api.cc` 第 8185 行：
```c++
     i::Isolate* isolate = new i::Isolate(false);
```
因此，我们正在调用 Isolate 构造函数（在 `src/isolate.cc` 中）。
```c++
    isolate->set_snapshot_blob(i::Snapshot::DefaultSnapshotBlob());
```
`api.cc`：
```c++
    isolate->Init(NULL);
    
    compilation_cache_ = new CompilationCache(this);
    context_slot_cache_ = new ContextSlotCache();
    descriptor_lookup_cache_ = new DescriptorLookupCache();
    unicode_cache_ = new UnicodeCache();
    inner_pointer_to_code_cache_ = new InnerPointerToCodeCache(this);
    global_handles_ = new GlobalHandles(this);
    eternal_handles_ = new EternalHandles();
    bootstrapper_ = new Bootstrapper(this);
    handle_scope_implementer_ = new HandleScopeImplementer(this);
    load_stub_cache_ = new StubCache(this, Code::LOAD_IC);
    store_stub_cache_ = new StubCache(this, Code::STORE_IC);
    materialized_object_store_ = new MaterializedObjectStore(this);
    regexp_stack_ = new RegExpStack();
    regexp_stack_->isolate_ = this;
    date_cache_ = new DateCache();
    call_descriptor_data_ =
      new CallInterfaceDescriptorData[CallDescriptors::NUMBER_OF_DESCRIPTORS];
    access_compiler_data_ = new AccessCompilerData();
    cpu_profiler_ = new CpuProfiler(this);
    heap_profiler_ = new HeapProfiler(heap());
    interpreter_ = new interpreter::Interpreter(this);
    compiler_dispatcher_ =
      new CompilerDispatcher(this, V8::GetCurrentPlatform(), FLAG_stack_size);
```
`src/builtins/builtins.cc` 是定义 Builtins 的地方。
TODO：理清这些宏的作用。

在 `src/v8.cc` 中，我们有几个检查，看传入的选项是否用于 `stress_run`，但由于我们没有传入任何此类标志，将遵循此代码路径，调用 `RunMain`：
```c++
    result = RunMain(isolate, argc, argv, last_run);
```
这最终将调用：
```c++
    options.isolate_sources[0].Execute(isolate);
```
这将调用 `SourceGroup::Execute(Isolate* isolate)`：
```c++
    // 使用所有其他参数作为要加载和运行的文件名。
    HandleScope handle_scope(isolate);
    Local<String> file_name = String::NewFromUtf8(isolate, arg, NewStringType::kNormal).ToLocalChecked();
    Local<String> source = ReadFile(isolate, arg);
    if (source.IsEmpty()) {
      printf("Error reading '%s'\n", arg);
      Shell::Exit(1);
    }
    Shell::options.script_executed = true;
    if (!Shell::ExecuteString(isolate, source, file_name, false, true)) {
      exception_was_thrown = true;
      break;
    }

    ScriptOrigin origin(name);
    if (compile_options == ScriptCompiler::kNoCompileOptions) {
      ScriptCompiler::Source script_source(source, origin);
      return ScriptCompiler::Compile(context, &script_source, compile_options);
    }
```
这将委托给 `ScriptCompiler(Local<Context>, Source* source, CompileOptions options)`：
```c++
    auto maybe = CompileUnboundInternal(isolate, source, options);
```
`CompileUnboundInternal`：
```c++
    result = i::Compiler::GetSharedFunctionInfoForScript(
        str, name_obj, line_offset, column_offset, source->resource_options,
        source_map_url, isolate->native_context(), NULL, &script_data, options,
        i::NOT_NATIVES_CODE);
```
`src/compiler.cc`：
```c++
    // 编译函数并将其添加到缓存中。
    ParseInfo parse_info(script);
    Zone compile_zone(isolate->allocator(), ZONE_NAME);
    CompilationInfo info(&compile_zone, &parse_info, Handle<JSFunction>::null());
```
回到 `src/compiler.cc-info.cc`：
```c++
    result = CompileToplevel(&info);

    (lldb) job *result
    0x17df0df309f1: [SharedFunctionInfo]
     - name = 0x1a7f12d82471 <String[0]: >
     - formal_parameter_count = 0
     - expected_nof_properties = 10
     - ast_node_count = 23
     - instance class name = #Object

     - code = 0x1d8484d3661 <Code: BUILTIN>
     - source code = function bajja(a, b, c) {
      var d = c - 100;
      return a + d * b;
    }

    var result = bajja(2, 2, 150);
    print(result);

     - anonymous expression
     - function token position = -1
     - start position = 0
     - end position = 114
     - no debug info
     - length = 0
     - optimized_code_map = 0x1a7f12d82241 <FixedArray[0]>
     - feedback_metadata = 0x17df0df30d09: [FeedbackMetadata]
     - length: 3
     - slot_count: 11
     Slot #0 LOAD_GLOBAL_NOT_INSIDE_TYPEOF_IC
     Slot #2 kCreateClosure
     Slot #3 LOAD_GLOBAL_NOT_INSIDE_TYPEOF_IC
     Slot #5 CALL_IC
     Slot #7 CALL_IC
     Slot #9 LOAD_GLOBAL_NOT_INSIDE_TYPEOF_IC

     - bytecode_array = 0x17df0df30c61
```
回到 `d8.cc`：
```c++
    maybe_result = script->Run(realm);
```
`src/api.cc`：
```c++
    auto fun = i::Handle<i::JSFunction>::cast(Utils::OpenHandle(this));

    (lldb) job *fun
    0x17df0df30e01: [Function]
     - map = 0x19cfe0003859 [FastProperties]
     - prototype = 0x17df0df043b1
     - elements = 0x1a7f12d82241 <FixedArray[0]> [FAST_HOLEY_ELEMENTS]
     - initial_map =
     - shared_info = 0x17df0df309f1 <SharedFunctionInfo>
     - name = 0x1a7f12d82471 <String[0]: >
     - formal_parameter_count = 0
     - context = 0x17df0df03bf9 <FixedArray[245]>
     - feedback vector cell = 0x17df0df30ed1 Cell for 0x17df0df30e49 <FixedArray[13]>
     - code = 0x1d8484d3661 <Code: BUILTIN>
     - properties = 0x1a7f12d82241 <FixedArray[0]> {
        #length: 0x2c35a5718089 <AccessorInfo> (const accessor descriptor)
        #name: 0x2c35a57180f9 <AccessorInfo> (const accessor descriptor)
        #arguments: 0x2c35a5718169 <AccessorInfo> (const accessor descriptor)
        #caller: 0x2c35a57181d9 <AccessorInfo> (const accessor descriptor)
        #prototype: 0x2c35a5718249 <AccessorInfo> (const accessor descriptor)

      }

    i::Handle<i::Object> receiver = isolate->global_proxy();
    Local<Value> result;
    has_pending_exception = !ToLocal<Value>(i::Execution::Call(isolate, fun, receiver, 0, nullptr), &result);
```
`src/execution.cc`

### Zone
直接取自 `src/zone/zone.h`：
```
// Zone 支持非常快速的小块内存分配。
// 这些块不能单独释放，相反，Zone 支持在一个快速操作中释放所有块。
// Zone 用于保存临时数据结构，如抽象语法树（AST），在编译后会被释放。
```

### V8 标志
```shell
    $ ./d8 --help
```

### d8
```console
    (lldb) br s -f d8.cc -l 2935

    return v8::Shell::Main(argc, argv);

    api.cc:6112
    i::ReadNatives();
    natives-external.cc
```

### v8::String::NewFromOneByte
当我第一次读到这个函数名时，我有点困惑，以为它与字符串的长度有关。但这里的 byte 是指组成字符串的字符类型。
例如，一个单字节字符将被重新解释为 `uint8_t`：
```c++
    const char* data

    reinterpret_cast<const uint8_t*>(data)
```

#### 任务
* `gdbinit` 已更新。检查是否有内容应移植到 `lldbinit`。

### 调用流程详解
本节将通过调用一个 Script 来了解 V8 中发生了什么。
我将使用 [run-scripts.cc](./run-scripts.cc) 作为示例。
```shell
    $ lldb -- ./run-scripts
    (lldb) br s -n main
```
我将逐步执行直到以下调用：
```c++
    script->Run(context).ToLocalChecked();
```
`Script::Run` 定义在 `api.cc` 中。
该函数中首先发生的是一个宏：
```c++
    PREPARE_FOR_EXECUTION_WITH_CONTEXT_IN_RUNTIME_CALL_STATS_SCOPE(
         "v8", 
         "V8.Execute", 
         context, 
         Script, 
         Run, 
         MaybeLocal<Value>(),
         InternalEscapableScope, 
    true);
    TRACE_EVENT_CALL_STATS_SCOPED(isolate, category, name);
    PREPARE_FOR_EXECUTION_GENERIC(isolate, context, class_name, function_name, \
        bailout_value, HandleScopeClass, do_callback);
```
那么预处理器会将其替换为什么：
```c++
    auto isolate = context.IsEmpty() ? i::Isolate::Current()                               : reinterpret_cast<i::Isolate*>(context->GetIsolate());
```
我暂时跳过 `TRACE_EVENT_CALL_STATS_SCOPED`。
`PREPARE_FOR_EXECUTION_GENERIC` 将被替换为：
```c++
    if (IsExecutionTerminatingCheck(isolate)) {                        \
      return bailout_value;                                            \
    }                                                                  \
    HandleScopeClass handle_scope(isolate);                            \
    CallDepthScope<do_callback> call_depth_scope(isolate, context);    \
    LOG_API(isolate, class_name, function_name);                       \
    ENTER_V8_DO_NOT_USE(isolate);                                      \
    bool has_pending_exception = false
```

```c++
    auto fun = i::Handle<i::JSFunction>::cast(Utils::OpenHandle(this));

    (lldb) job *fun
    0x33826912c021: [Function]
     - map = 0x1d0656c03599 [FastProperties]
     - prototype = 0x338269102e69
     - elements = 0x35190d902241 <FixedArray[0]> [FAST_HOLEY_ELEMENTS]
     - initial_map =
     - shared_info = 0x33826912bc11 <SharedFunctionInfo>
     - name = 0x35190d902471 <String[0]: >
     - formal_parameter_count = 0
     - context = 0x338269102611 <FixedArray[265]>
     - feedback vector cell = 0x33826912c139 <Cell value= 0x33826912c069 <FixedArray[24]>>
     - code = 0x1319e25fcf21 <Code BUILTIN>
     - properties = 0x35190d902241 <FixedArray[0]> {
        #length: 0x2e9d97ce68b1 <AccessorInfo> (const accessor descriptor)
        #name: 0x2e9d97ce6921 <AccessorInfo> (const accessor descriptor)
        #arguments: 0x2e9d97ce6991 <AccessorInfo> (const accessor descriptor)
        #caller: 0x2e9d97ce6a01 <AccessorInfo> (const accessor descriptor)
        #prototype: 0x2e9d97ce6a71 <AccessorInfo> (const accessor descriptor)
     }
```
`i::JSFunction` 的代码是在 `src/api.h` 中生成的。让我们仔细看看。
```c++
    #define DECLARE_OPEN_HANDLE(From, To) \
      static inline v8::internal::Handle<v8::internal::To> \
      OpenHandle(const From* that, bool allow_empty_handle = false);

    OPEN_HANDLE_LIST(DECLARE_OPEN_HANDLE)
```
`OPEN_HANDLE_LIST` 看起来像这样：
```c++
    #define OPEN_HANDLE_LIST(V)                    \
    ....
    V(Script, JSFunction)                        \ 
```
因此，将其展开为 `JSFunction` 后应变为：
```c++
      static inline v8::internal::Handle<v8::internal::JSFunction> \
        OpenHandle(const Script* that, bool allow_empty_handle = false);
```
因此将有一个名为 `OpenHandle` 的函数，它接受一个指向 `Script` 的 const 指针。

在 `src/api.h` 稍下方还有另一个宏：
```c++
    OPEN_HANDLE_LIST(MAKE_OPEN_HANDLE)
```
`MAKE_OPEN_HANDLE`：
```c++
    #define MAKE_OPEN_HANDLE(From, To)
      v8::internal::Handle<v8::internal::To> Utils::OpenHandle( 
      const v8::From* that, bool allow_empty_handle) {         
      return v8::internal::Handle<v8::internal::To>(                         
        reinterpret_cast<v8::internal::Address*>(const_cast<v8::From*>(that))); 
      }
```
请记住 `JSFunction` 包含在 `OPEN_HANDLE_LIST` 中，因此在预处理器处理此头文件后，源代码中将出现以下内容：
一个具体的例子看起来像这样：
```c++
v8::internal::Handle<v8::internal::JSFunction> Utils::OpenHandle(
    const v8::Script* that, bool allow_empty_handle) {
  return v8::internal::Handle<v8::internal::JSFunction>(
      reinterpret_cast<v8::internal::Address*>(const_cast<v8::Script*>(that))); }
```
你可以使用以下命令检查预处理器的输出：
```console
$ clang++ -I./out/x64.release/gen -I. -I./include -E src/api/api-inl.h > api-inl.output
```

那么 `JSFunction` 是在哪里声明的？
它定义在 `objects.h` 中。

## Ignition 解释器
用户 JavaScript 也需要为其生成字节码，它们也使用 C++ DSL，并像 Builtins 一样使用 `CodeStubAssembler -> CodeAssembler -> RawMachineAssembler`。

## C++ 领域特定语言 (DSL)

#### 构建失败
在 rebase 之后，我遇到了以下问题：
```shell
    $ ninja -C out/Debug chrome
    ninja: Entering directory `out/Debug'
    ninja: error: '../../chrome/renderer/resources/plugins/plugin_delay.html', needed by 'gen/chrome/grit/renderer_resources.h', missing and no known rule to make it
```
“解决方案”是删除 `out` 目录并重新构建。

### 任务
要寻找合适的任务，你可以在 [bugs.chromium.org](https://bugs.chromium.org/p/v8/issues/list?can=2&q=label%3AHelpWanted+&colspec=ID+Type+Status+Priority+Owner+Summary+HW+OS+Component+Stars&x=priority&y=owner&cells=ids) 使用 `label:HelpWanted`。

### OpenHandle
此调用是做什么的：
```c++
    Utils::OpenHandle(*(source->source_string));

    OPEN_HANDLE_LIST(MAKE_OPEN_HANDLE)
```
这是定义在 `src/api.h` 中的一个宏：
```c++
    #define MAKE_OPEN_HANDLE(From, To)                                             \
      v8::internal::Handle<v8::internal::To> Utils::OpenHandle(                    \
          const v8::From* that, bool allow_empty_handle) {                         \
      DCHECK(allow_empty_handle || that != NULL);                                \
      DCHECK(that == NULL ||                                                     \
           (*reinterpret_cast<v8::internal::Object* const*>(that))->Is##To()); \
      return v8::internal::Handle<v8::internal::To>(                             \
          reinterpret_cast<v8::internal::To**>(const_cast<v8::From*>(that)));    \
    }
```
Builtins 曾被编译并存储在快照中，但现在看来它们被放置在 `out.gn/learning/gen/embedded.cc` 中，并与编译产生的对象文件结合生成 `libv8.dylib`。
V8 有一个名为 `v8_enable_embedded_builtins` 的配置选项，在这种情况下，`embedded.cc` 将被添加到源文件列表中。这是在 `BUILD.gn` 和 `v8_snapshot` 目标中完成的。如果 `v8_enable_embedded_builtins` 为 false，则将包含 `src/snapshot/embedded-empty.cc`。这两个文件都有以下函数：
```c++
const uint8_t* DefaultEmbeddedBlob()
uint32_t DefaultEmbeddedBlobSize()

#ifdef V8_MULTI_SNAPSHOTS
const uint8_t* TrustedEmbeddedBlob()
uint32_t TrustedEmbeddedBlobSize()
#endif
```
这些函数由 `isolate.cc` 使用并声明为 `extern`：
```c++
extern const uint8_t* DefaultEmbeddedBlob();
extern uint32_t DefaultEmbeddedBlobSize();
```
在 `Isolate::Isolate` 中可以看到 `DefaultEmbeddedBlob` 的用法，它在那里设置嵌入式 Blob：
```c++
SetEmbeddedBlob(DefaultEmbeddedBlob(), DefaultEmbeddedBlobSize());
```
让我们在那里设置一个断点，看看它是否为空。
```console
(lldb) expr v8_embedded_blob_size_
(uint32_t) $0 = 4021088
```
可以看到我们没有使用空的。

在 `src/snapshot/deserializer.cc`（第 552 行）中，我们有一个对 `embedded_blob()` 的检查：
```c++
  CHECK_NOT_NULL(isolate->embedded_blob());
  EmbeddedData d = EmbeddedData::FromBlob();
  Address address = d.InstructionStartOfBuiltin(builtin_index);
```
`EmbeddedData` 可以在 `src/snapshot/snapshot.h` 中找到，实现在 `snapshot-common.cc` 中。
```c++
Address EmbeddedData::InstructionStartOfBuiltin(int i) const {
  const struct Metadata* metadata = Metadata();
  const uint8_t* result = RawData() + metadata[i].instructions_offset;
  return reinterpret_cast<Address>(result);
}
```
```console
(lldb) expr *metadata
(const v8::internal::EmbeddedData::Metadata) $7 = (instructions_offset = 0, instructions_length = 1464)
```
```c++
  struct Metadata {
    // Blob 布局信息。
    uint32_t instructions_offset;
    uint32_t instructions_length;
  };
```
```console
(lldb) expr *this
(v8::internal::EmbeddedData) $10 = (data_ = "\xffffffdc\xffffffc0\xffffff88'"y[\xffffffd6", size_ = 4021088)
(lldb) expr metadata[i]
(const v8::internal::EmbeddedData::Metadata) $8 = (instructions_offset = 0, instructions_length = 1464)
```
那么，我们是否可以验证这些信息在 `.text` 段中？
```console
(lldb) expr result
(const uint8_t *) $13 = 0x0000000101b14ee0 "UH\x89jH\x83(H\x89UH\x16H\x89}HuHEH\x89UH\x83
(lldb) image lookup --address 0x0000000101b14ee0 --verbose
      Address: libv8.dylib[0x00000000019cdee0] (libv8.dylib.__TEXT.__text + 27054464)
      Summary: libv8.dylib`v8_Default_embedded_blob_ + 7072
       Module: file = "/Users/danielbevenius/work/google/javascript/v8/out.gn/learning/libv8.dylib", arch = "x86_64"
       Symbol: id = {0x0004b596}, range = [0x0000000101b13340-0x0000000101ee8ea0), name="v8_Default_embedded_blob_"
```
因此我们得到的是一个指向返回的 `.text` 段的指针：
```console
(lldb) memory read -f x -s 1 -c 13 0x0000000101b14ee0
0x101b14ee0: 0x55 0x48 0x89 0xe5 0x6a 0x18 0x48 0x83
0x101b14ee8: 0xec 0x28 0x48 0x89 0x55
```
我们可以将其与 `out.gn/learning/gen/embedded.cc` 进行比较：
```c++
V8_EMBEDDED_TEXT_HEADER(v8_Default_embedded_blob_)
__asm__(
  ...
  ".byte 0x55,0x48,0x89,0xe5,0x6a,0x18,0x48,0x83,0xec,0x28,0x48,0x89,0x55\n"
  ...
);
```
宏 `V8_EMBEDDED_TEXT_HEADER` 可以在 `src/snapshot/macros.h` 中找到：
```c++
#define V8_EMBEDDED_TEXT_HEADER(LABEL)         \
  __asm__(V8_ASM_DECLARE(#LABEL)               \
          ".csect " #LABEL "[DS]\n"            \
          #LABEL ":\n"                         \
          ".llong ." #LABEL ", TOC[tc0], 0\n"  \
          V8_ASM_TEXT_SECTION                  \
          "." #LABEL ":\n");

define V8_ASM_DECLARE(NAME) ".private_extern " V8_ASM_MANGLE_LABEL NAME "\n"
#define V8_ASM_MANGLE_LABEL "_"
#define V8_ASM_TEXT_SECTION ".csect .text[PR]\n"
```
预处理器会将其展开为：
```c++
  __asm__(".private_extern " _ v8_Default_embedded_blob_ "\n"
          ".csect " v8_Default_embedded_blob_ "[DS]\n"
          v8_Default_embedded_blob_ ":\n"
          ".llong ." v8_Default_embedded_blob_ ", TOC[tc0], 0\n"
          ".csect .text[PR]\n"
          "." v8_Default_embedded_blob_ ":\n");
  __asm__(
    ...
    ".byte 0x55,0x48,0x89,0xe5,0x6a,0x18,0x48,0x83,0xec,0x28,0x48,0x89,0x55\n"
    ...
  );
```

回到 `src/snapshot/deserializer.cc`：
```c++
  Address address = d.InstructionStartOfBuiltin(builtin_index);
  CHECK_NE(kNullAddress, address);
  if (RelocInfo::OffHeapTargetIsCodedSpecially()) {
    // 在我们的例子中为 false，因此跳过这里的代码
  } else {
    MaybeObject* o = reinterpret_cast<MaybeObject*>(address);
    UnalignedCopy(current, &o);
    current++;
  }
  break;
```

### print-code
```console
$ ./d8 -print-bytecode  -print-code sample.js 
[generated bytecode for function:  (0x2a180824ffbd <SharedFunctionInfo>)]
Parameter count 1
Register count 5
Frame size 40
         0x2a1808250066 @    0 : 12 00             LdaConstant [0]
         0x2a1808250068 @    2 : 26 f9             Star r2
         0x2a180825006a @    4 : 27 fe f8          Mov <closure>, r3
         0x2a180825006d @    7 : 61 32 01 f9 02    CallRuntime [DeclareGlobals], r2-r3
         0x2a1808250072 @   12 : 0b                LdaZero 
         0x2a1808250073 @   13 : 26 fa             Star r1
         0x2a1808250075 @   15 : 0d                LdaUndefined 
         0x2a1808250076 @   16 : 26 fb             Star r0
         0x2a1808250078 @   18 : 00 0c 10 27       LdaSmi.Wide [10000]
         0x2a180825007c @   22 : 69 fa 00          TestLessThan r1, [0]
         0x2a180825007f @   25 : 9a 1c             JumpIfFalse [28] (0x2a180825009b @ 53)
         0x2a1808250081 @   27 : a7                StackCheck 
         0x2a1808250082 @   28 : 13 01 01          LdaGlobal [1], [1]
         0x2a1808250085 @   31 : 26 f9             Star r2
         0x2a1808250087 @   33 : 0c 02             LdaSmi [2]
         0x2a1808250089 @   35 : 26 f7             Star r4
         0x2a180825008b @   37 : 5e f9 fa f7 03    CallUndefinedReceiver2 r2, r1, r4, [3]
         0x2a1808250090 @   42 : 26 fb             Star r0
         0x2a1808250092 @   44 : 25 fa             Ldar r1
         0x2a1808250094 @   46 : 4c 05             Inc [5]
         0x2a1808250096 @   48 : 26 fa             Star r1
         0x2a1808250098 @   50 : 8a 20 00          JumpLoop [32], [0] (0x2a1808250078 @ 18)
         0x2a180825009b @   53 : 25 fb             Ldar r0
         0x2a180825009d @   55 : ab                Return 
Constant pool (size = 2)
0x2a1808250035: [FixedArray] in OldSpace
 - map: 0x2a18080404b1 <Map>
 - length: 2
           0: 0x2a180824ffe5 <FixedArray[2]>
           1: 0x2a180824ff61 <String[#9]: something>
Handler Table (size = 0)
Source Position Table (size = 0)
[generated bytecode for function: something (0x2a180824fff5 <SharedFunctionInfo something>)]
Parameter count 3
Register count 0
Frame size 0
         0x2a18082501ba @    0 : 25 02             Ldar a1
         0x2a18082501bc @    2 : 34 03 00          Add a0, [0]
         0x2a18082501bf @    5 : ab                Return 
Constant pool (size = 0)
Handler Table (size = 0)
Source Position Table (size = 0)
--- Raw source ---
function something(x, y) {
  return x + y
}
for (let i = 0; i < 10000; i++) {
  something(i, 2);
}


--- Optimized code ---
optimization_id = 0
source_position = 0
kind = OPTIMIZED_FUNCTION
stack_slots = 14
compiler = turbofan
address = 0x108400082ae1

Instructions (size = 536)
0x108400082b20     0  488d1df9ffffff REX.W leaq rbx,[rip+0xfffffff9]
0x108400082b27     7  483bd9         REX.W cmpq rbx,rcx
0x108400082b2a     a  7418           jz 0x108400082b44  <+0x24>
0x108400082b2c     c  48ba6800000000000000 REX.W movq rdx,0x68
0x108400082b36    16  49bae0938c724b560000 REX.W movq r10,0x564b728c93e0  (Abort)    ;; off heap target
0x108400082b40    20  41ffd2         call r10
0x108400082b43    23  cc             int3l
0x108400082b44    24  8b59d0         movl rbx,[rcx-0x30]
0x108400082b47    27  4903dd         REX.W addq rbx,r13
0x108400082b4a    2a  f6430701       testb [rbx+0x7],0x1
0x108400082b4e    2e  740d           jz 0x108400082b5d  <+0x3d>
0x108400082b50    30  49bae0f781724b560000 REX.W movq r10,0x564b7281f7e0  (CompileLazyDeoptimizedCode)    ;; off heap target
0x108400082b5a    3a  41ffe2         jmp r10
0x108400082b5d    3d  55             push rbp
0x108400082b5e    3e  4889e5         REX.W movq rbp,rsp
0x108400082b61    41  56             push rsi
0x108400082b62    42  57             push rdi
0x108400082b63    43  48ba4200000000000000 REX.W movq rdx,0x42
0x108400082b6d    4d  4c8b15c4ffffff REX.W movq r10,[rip+0xffffffc4]
0x108400082b74    54  41ffd2         call r10
0x108400082b77    57  cc             int3l
0x108400082b78    58  4883ec18       REX.W subq rsp,0x18
0x108400082b7c    5c  488975a0       REX.W movq [rbp-0x60],rsi
0x108400082b80    60  488b4dd0       REX.W movq rcx,[rbp-0x30]
0x108400082b84    64  f6c101         testb rcx,0x1
0x108400082b87    67  0f8557010000   jnz 0x108400082ce4  <+0x1c4>
0x108400082b8d    6d  81f9204e0000   cmpl rcx,0x4e20
0x108400082b93    73  0f8c0b000000   jl 0x108400082ba4  <+0x84>
0x108400082b99    79  488b45d8       REX.W movq rax,[rbp-0x28]
0x108400082b9d    7d  488be5         REX.W movq rsp,rbp
0x108400082ba0    80  5d             pop rbp
0x108400082ba1    81  c20800         ret 0x8
0x108400082ba4    84  493b6560       REX.W cmpq rsp,[r13+0x60] (external value (StackGuard::address_of_jslimit()))
0x108400082ba8    88  0f8669000000   jna 0x108400082c17  <+0xf7>
0x108400082bae    8e  488bf9         REX.W movq rdi,rcx
0x108400082bb1    91  d1ff           sarl rdi, 1
0x108400082bb3    93  4c8bc7         REX.W movq r8,rdi
0x108400082bb6    96  4183c002       addl r8,0x2
0x108400082bba    9a  0f8030010000   jo 0x108400082cf0  <+0x1d0>
0x108400082bc0    a0  83c701         addl rdi,0x1
0x108400082bc3    a3  0f8033010000   jo 0x108400082cfc  <+0x1dc>
0x108400082bc9    a9  e921000000     jmp 0x108400082bef  <+0xcf>
0x108400082bce    ae  6690           nop
0x108400082bd0    b0  488bcf         REX.W movq rcx,rdi
0x108400082bd3    b3  83c102         addl rcx,0x2
0x108400082bd6    b6  0f802c010000   jo 0x108400082d08  <+0x1e8>
0x108400082bdc    bc  4c8bc7         REX.W movq r8,rdi
0x108400082bdf    bf  4183c001       addl r8,0x1
0x108400082be3    c3  0f802b010000   jo 0x108400082d14  <+0x1f4>
0x108400082be9    c9  498bf8         REX.W movq rdi,r8
0x108400082bec    cc  4c8bc1         REX.W movq r8,rcx
0x108400082bef    cf  81ff10270000   cmpl rdi,0x2710
0x108400082bf5    d5  0f8d0b000000   jge 0x108400082c06  <+0xe6>
0x108400082bfb    db  493b6560       REX.W cmpq rsp,[r13+0x60] (external value (StackGuard::address_of_jslimit()))
0x108400082bff    df  77cf           ja 0x108400082bd0  <+0xb0>
0x108400082c01    e1  e943000000     jmp 0x108400082c49  <+0x129>
0x108400082c06    e6  498bc8         REX.W movq rcx,r8
0x108400082c09    e9  4103c8         addl rcx,r8
0x108400082c0c    ec  0f8061000000   jo 0x108400082c73  <+0x153>
0x108400082c12    f2  488bc1         REX.W movq rax,rcx
0x108400082c15    f5  eb86           jmp 0x108400082b9d  <+0x7d>
0x108400082c17    f7  33c0           xorl rax,rax
0x108400082c19    f9  48bef50c240884100000 REX.W movq rsi,0x108408240cf5    ;; object: 0x108408240cf5 <NativeContext[261]>
0x108400082c23   103  48bb101206724b560000 REX.W movq rbx,0x564b72061210    ;; external reference (Runtime::StackGuard)
0x108400082c2d   10d  488bf8         REX.W movq rdi,rax
0x108400082c30   110  4c8bc6         REX.W movq r8,rsi
0x108400082c33   113  49ba2089a3724b560000 REX.W movq r10,0x564b72a38920  (CEntry_Return1_DontSaveFPRegs_ArgvOnStack_NoBuiltinExit)    ;; off heap target
0x108400082c3d   11d  41ffd2         call r10
0x108400082c40   120  488b4dd0       REX.W movq rcx,[rbp-0x30]
0x108400082c44   124  e965ffffff     jmp 0x108400082bae  <+0x8e>
0x108400082c49   129  48897da8       REX.W movq [rbp-0x58],rdi
0x108400082c4d   12d  488b1dd1ffffff REX.W movq rbx,[rip+0xffffffd1]
0x108400082c54   134  33c0           xorl rax,rax
0x108400082c56   136  48bef50c240884100000 REX.W movq rsi,0x108408240cf5    ;; object: 0x108408240cf5 <NativeContext[261]>
0x108400082c60   140  4c8b15ceffffff REX.W movq r10,[rip+0xffffffce]
0x108400082c67   147  41ffd2         call r10
0x108400082c6a   14a  488b7da8       REX.W movq rdi,[rbp-0x58]
0x108400082c6e   14e  e95dffffff     jmp 0x108400082bd0  <+0xb0>
0x108400082c73   153  48b968ea2f744b560000 REX.W movq rcx,0x564b742fea68    ;; external reference (Heap::NewSpaceAllocationTopAddress())
0x108400082c7d   15d  488b39         REX.W movq rdi,[rcx]
0x108400082c80   160  4c8d4f0c       REX.W leaq r9,[rdi+0xc]
0x108400082c84   164  4c8945b0       REX.W movq [rbp-0x50],r8
0x108400082c88   168  49bb70ea2f744b560000 REX.W movq r11,0x564b742fea70    ;; external reference (Heap::NewSpaceAllocationLimitAddress())
0x108400082c92   172  4d390b         REX.W cmpq [r11],r9
0x108400082c95   175  0f8721000000   ja 0x108400082cbc  <+0x19c>
0x108400082c9b   17b  ba0c000000     movl rdx,0xc
0x108400082ca0   180  49ba200282724b560000 REX.W movq r10,0x564b72820220  (AllocateRegularInYoungGeneration)    ;; off heap target
0x108400082caa   18a  41ffd2         call r10
0x108400082cad   18d  488d78ff       REX.W leaq rdi,[rax-0x1]
0x108400082cb1   191  488b0dbdffffff REX.W movq rcx,[rip+0xffffffbd]
0x108400082cb8   198  4c8b45b0       REX.W movq r8,[rbp-0x50]
0x108400082cbc   19c  4c8d4f0c       REX.W leaq r9,[rdi+0xc]
0x108400082cc0   1a0  4c8909         REX.W movq [rcx],r9
0x108400082cc3   1a3  488d4f01       REX.W leaq rcx,[rdi+0x1]
0x108400082cc7   1a7  498bbd40010000 REX.W movq rdi,[r13+0x140] (root (heap_number_map))
0x108400082cce   1ae  8979ff         movl [rcx-0x1],rdi
0x108400082cd1   1b1  c4c1032ac0     vcvtlsi2sd xmm0,xmm15,r8
0x108400082cd6   1b6  c5fb114103     vmovsd [rcx+0x3],xmm0
0x108400082cdb   1bb  488bc1         REX.W movq rax,rcx
0x108400082cde   1be  e9bafeffff     jmp 0x108400082b9d  <+0x7d>
0x108400082ce3   1c3  90             nop
0x108400082ce4   1c4  49c7c500000000 REX.W movq r13,0x0
0x108400082ceb   1cb  e850f30300     call 0x1084000c2040     ;; eager deoptimization bailout
0x108400082cf0   1d0  49c7c501000000 REX.W movq r13,0x1
0x108400082cf7   1d7  e844f30300     call 0x1084000c2040     ;; eager deoptimization bailout
0x108400082cfc   1dc  49c7c502000000 REX.W movq r13,0x2
0x108400082d03   1e3  e838f30300     call 0x1084000c2040     ;; eager deoptimization bailout
0x108400082d08   1e8  49c7c503000000 REX.W movq r13,0x3
0x108400082d0f   1ef  e82cf30300     call 0x1084000c2040     ;; eager deoptimization bailout
0x108400082d14   1f4  49c7c504000000 REX.W movq r13,0x4
```
```shell
g++ -MMD -MF obj/test/cctest/cctest_sources/test-global-handles.o.d 
...
In file included from ../../include/v8-inspector.h:14,
                 from ../../src/execution/isolate.h:15,
                 from ../../src/api/api.h:10,
                 from ../../src/api/api-inl.h:8,
                 from ../../test/cctest/test-global-handles.cc:28:
../../include/v8.h:
In instantiation of ‘void v8::PersistentBase<T>::SetWeak(
    P*,
    typename v8::WeakCallbackInfo<P>::Callback,
    v8::WeakCallbackType)
[with 
  P = v8::Global<v8::Object>; 
  T = v8::Object;
  typename v8::WeakCallbackInfo<P>::Callback =
  void (*)(const v8::WeakCallbackInfo<v8::Global<v8::Object> >&)
]’:
../../test/cctest/test-global-handles.cc:292:47:   required from here
../../include/v8.h:10750:16: warning:
cast between incompatible function types from
‘v8::WeakCallbackInfo<v8::Global<v8::Object> >::Callback’ {aka
‘void (*)(const v8::WeakCallbackInfo<v8::Global<v8::Object> >&)’} to 
‘Callback’ {aka ‘void (*)(const v8::WeakCallbackInfo<void>&)’}
[-Wcast-function-type]
10750 |                reinterpret_cast<Callback>(callback), type);
      |                ^~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
```
此提交建议专门为 GCC 添加一个 pragma 以抑制此警告。这样做的动机是 Node.js 构建中有很多此类警告，但这些警告已通过在包含 `v8.h` [1] 的周围添加类似的 pragma 被抑制了。

[1] https://github.com/nodejs/node/blob/331d63624007be4bf49d6d161bdef2b5e540affa/src/node.h#L63-L70

```console
$ 
In file included from persistent-obj.cc:8:
/home/danielbevenius/work/google/v8_src/v8/include/v8.h: In instantiation of ‘void v8::PersistentBase<T>::SetWeak(P*, typename v8::WeakCallbackInfo<P>::Callback, v8::WeakCallbackType) [with P = Something; T = v8::Object; typename v8::WeakCallbackInfo<P>::Callback = void (*)(const v8::WeakCallbackInfo<Something>&)]’:

persistent-obj.cc:57:38:   required from here
/home/danielbevenius/work/google/v8_src/v8/include/v8.h:10750:16: warning: cast between incompatible function types from ‘v8::WeakCallbackInfo<Something>::Callback’ {aka ‘void (*)(const v8::WeakCallbackInfo<Something>&)’} to ‘Callback’ {aka ‘void (*)(const v8::WeakCallbackInfo<void>&)’} [-Wcast-function-type]
10750 |                reinterpret_cast<Callback>(callback), type);
      |                ^~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
```
目前，我们已经添加了一个 pragma 来避免 Node.js 中的这个警告，但我们希望在 V8 中以及更接近导致该警告的实际代码处添加它。在 Node 中，我们必须在头文件上设置 pragma。
```c++
template <class T>
template <typename P>
V8_INLINE void PersistentBase<T>::SetWeak(
    P* parameter,
    typename WeakCallbackInfo<P>::Callback callback,
    WeakCallbackType type) {
  typedef typename WeakCallbackInfo<void>::Callback Callback;
  V8::MakeWeak(reinterpret_cast<internal::Address*>(this->val_), parameter,
               reinterpret_cast<Callback>(callback), type);
}
```
注意第二个参数是 `typename WeakCallbackInfo<P>::Callback`，它是一个 typedef：
```c++
  typedef void (*Callback)(const WeakCallbackInfo<T>& data);
```
这是 `Callback` 的函数声明，它是一个接受指向 const `WeakCallbackInfo<T>` 的引用并返回 void 的函数。所以我们可以这样定义它：
```c++
void WeakCallback(const v8::WeakCallbackInfo<Something>& data) {
  Something* obj = data.GetParameter();
  std::cout << "in make weak callback..." << '\n';
}
```
然后尝试将其转换为：
```c++
  typedef typename v8::WeakCallbackInfo<void>::Callback Callback;
  Callback cb = reinterpret_cast<Callback>(WeakCallback);
```
这样做是因为 `V8::MakeWeak` 具有以下签名：
```c++
void V8::MakeWeak(i::Address* location, void* parameter,
                  WeakCallbackInfo<void>::Callback weak_callback,
                  WeakCallbackType type) {
  i::GlobalHandles::MakeWeak(location, parameter, weak_callback, type);
}
```

### gdb 警告
```console
warning: Could not find DWO CU obj/v8_compiler/common-node-cache.dwo(0x42b8adb87d74d56b) referenced by CU at offset 0x206f7 [in module /home/danielbevenius/work/google/learning-v8/hello-world]
```
这可以通过向 gdb 指定 `--cd` 参数来解决：
```console
$ gdb --cd=/home/danielbevenius/work/google/v8_src/v8/out/x64.release --args /home/danielbevenius/work/google/learning-v8/hello-world
```

### 使用 g++ 构建
更新 `args.gn` 以包含：
```
is_clang = false
```
接下来，我在尝试编译时遇到了以下错误：
```console
$ ninja -v -C out/x64.release/ obj/test/cctest/cctest_sources/test-global-handles.o
ux/debian_sid_amd64-sysroot -fexceptions -frtti -c ../../src/torque/instance-type-generator.cc -o obj/torque_base/instance-type-generator.o
In file included from /usr/include/c++/9/bits/stl_algobase.h:59,
                 from /usr/include/c++/9/memory:62,
                 from ../../src/torque/implementation-visitor.h:8,
                 from ../../src/torque/instance-type-generator.cc:5:
/usr/include/c++/9/x86_64-redhat-linux/bits/c++config.h:3:10: fatal error: bits/wordsize.h: No such file or directory
    3 | #include <bits/wordsize.h>
      |          ^~~~~~~~~~~~~~~~~
compilation terminated.
ninja: build stopped: subcommand failed.
```
```console
$ export CPATH=/usr/include
```

```console
third_party/binutils/Linux_x64/Release/bin/ld.gold: error: cannot open /usr/lib64/libatomic.so.1.2.0: No such file or directory
```
```console
$ sudo dnf install -y libatomic
```
由于警告，我仍然收到错误，但我尝试使用以下设置进行构建：
```
treat_warnings_as_errors = false
```
让我们看看效果如何。我还必须通过禁用 gold 来使用 gnu 链接器：
```console
use_gold = false
```

### CodeStubAssembler
其历史是 JavaScript Builtins 过去是用汇编编写的，这提供了非常好的性能，但使将 V8 移植到不同架构变得更加困难，因为这些 Builtins 必须为每个支持的架构提供特定的实现，因此它的扩展性不是很好。随着 JavaScript 规范中功能的增加，必须支持新功能意味着必须为所有平台实现它们，这使得跟上并交付这些新功能变得困难。

目标是拥有手写汇编的性能，但不必为每个平台编写它。因此，在 Turbofan 后端之上构建了一种可移植的汇编语言。这是一个生成 Turbofan 机器级 IR 的 API。此 IR 可由 Turbofan 使用，在所有平台上生成非常好的机器代码。因此，“只需”实现一个组件/函数/功能（不确定如何称呼它），然后就可以在所有平台上使用。他们不再需要维护所有那些手写的汇编。

明确地说， CSA 是一个 C++ API，用于生成 IR，然后将其编译为目标指令集架构的机器代码。

### Torque
[Torque](https://v8.dev/docs/torque) 是一种 DSL 语言，旨在避免直接使用 `CodeStubAssembler`（它仍在幕后被使用）。这种语言是静态类型的、垃圾回收的，并且与 JavaScript 兼容。

JavaScript 标准库以前在 V8 中是使用手写汇编实现的。但正如我们在上一节中提到的，这无法扩展。

它也可以用 JavaScript 编写，我想过去确实是这样做的，但这有一些问题，因为 Builtins 需要预热时间才能变得优化，还存在 Monkey-patching 和无意中暴露 VM 内部机制的问题。

Torque 是在构建时运行的吗？我想是的，因为它必须生成 C++ 代码。

`torque.cc` 中有一个 main 函数，它将被构建为一个可执行文件。
```console
$ ./out/x64.release_gcc/torque --help
Unexpected command-line argument "--help", expected a .tq file.
```

由 Torque 处理的文件在 `BUILD.gc` 的 `torque_files` 部分定义。还有一个名为 `run_torque` 的模板。我注意到这个模板和 GN 中的其他模板使用脚本 `tools/run.py`。这显然是因为 GN 目前只能执行脚本，而这个脚本的作用是使用 Python 创建一个带有传入参数的子进程：
```console
$ gn help action
```
模板是在 GN 中重用代码的一种方式。

有一个 make 目标可以显示 Torque 生成的内容：
```console
$ make torque-example
```
这将在当前目录中创建一个名为 `gen/torque-generated` 的目录。注意该目录包含 C++ 头文件和源文件。

它以 [torque-example.tq](./torque-example.tq) 作为输入。对于此文件，将生成以下头文件：
```c++
#ifndef V8_GEN_TORQUE_GENERATED_TORQUE_EXAMPLE_TQ_H_                            
#define V8_GEN_TORQUE_GENERATED_TORQUE_EXAMPLE_TQ_H_                            
                                                                                
#include "src/builtins/builtins-promise.h"                                      
#include "src/compiler/code-assembler.h"                                        
#include "src/codegen/code-stub-assembler.h"                                    
#include "src/utils/utils.h"                                                    
#include "torque-generated/field-offsets-tq.h"                                  
#include "torque-generated/csa-types-tq.h"                                      
                                                                                
namespace v8 {                                                                  
namespace internal {                                                            
                                                                                
void HelloWorld_0(compiler::CodeAssemblerState* state_);                        

}  // namespace internal                                                        
}  // namespace v8                                                              
                                                                                
#endif  // V8_GEN_TORQUE_GENERATED_TORQUE_EXAMPLE_TQ_H_
```
这只是为了展示生成的文件，并明确 Torque 将生成这些文件，然后在 V8 构建期间进行编译。所以，让我们尝试将 `example-torque.tq` 复制到 `v8/src/builtins` 目录。
```console
$ cp torque-example.tq ../v8_src/v8/src/builtins/
```
这不足以将其包含在构建中，我们必须更新 `BUILD.gn` 并将此文件添加到 `torque_files` 列表中。运行构建后，我们可以看到生成了一个名为 `src/builtins/torque-example-tq-csa.h` 的文件以及一个 `.cc` 文件。

为了理解这是如何工作的，我将使用 https://v8.dev/docs/torque-builtins 作为起点：
```
  transitioning javascript builtin                                              
  MathIs42(js-implicit context: NativeContext, receiver: JSAny)(x: JSAny): Boolean {
    const number: Number = ToNumber_Inline(x);                                  
    typeswitch (number) {                                                       
      case (smi: Smi): {                                                        
        return smi == 42 ? True : False;                                        
      }                                                                         
      case (heapNumber: HeapNumber): {                                          
        return Convert<float64>(heapNumber) == 42 ? True : False;               
      }                                                                         
    }                                                                           
  }                   
```
这已更新以适用于最新的 V8 版本。

接下来，我们需要更新 `src/init/bootstrappers.cc` 以在 math 对象上添加/安装此函数：
```c++
  SimpleInstallFunction(isolate_, math, "is42", Builtins::kMathIs42, 1, true);
```
之后我们需要重新构建 V8：
```console
$ env CPATH=/usr/include ninja -v -C out/x64.release_gcc
```
```console
$ d8
d8> Math.is42(42)
true
d8> Math.is42(2)
false
```

如果我们查看 Torque 在 `out/x64.release_gcc/gen/torque-generated/src/builtins/math-tq-csa.cc` 中生成的代码（我们可以使用以下命令通过预处理器运行它）：
```console
$ clang++ --sysroot=build/linux/debian_sid_amd64-sysroot -isystem=./buildtools/third_party/libc++/trunk/include -isystem=buildtools/third_party/libc++/trunk/include -I. -E out/x64.release_gcc/gen/torque-generated/src/builtins/math-tq-csa.cc > math.cc.pp
```
如果我们打开 `math.cc.pp` 并搜索 `Is42`，我们可以找到：
```c++
class MathIs42Assembler : public CodeStubAssembler {                            
 public:                                                                        
  using Descriptor = Builtin_MathIs42_InterfaceDescriptor;                      
  explicit MathIs42Assembler(compiler::CodeAssemblerState* state) : CodeStubAssembler(state) {}
  void GenerateMathIs42Impl();                                                  
  Node* Parameter(Descriptor::ParameterIndices index) {                         
    return CodeAssembler::Parameter(static_cast<int>(index));                   
  }                                                                             
};                                                                              
                                                                                
void Builtins::Generate_MathIs42(compiler::CodeAssemblerState* state) {         
  MathIs42Assembler assembler(state);                                           
  state->SetInitialDebugInformation("MathIs42", "out/x64.release_gcc/gen/torque-generated/src/builtins/math-tq-csa.cc", 2121);
  if (Builtins::KindOf(Builtins::kMathIs42) == Builtins::TFJ) {                 
    assembler.PerformStackCheck(assembler.GetJSContextParameter());             
  }                                                                             
  assembler.GenerateMathIs42Impl();                                             
}                                                                               
                                                                                
void MathIs42Assembler::GenerateMathIs42Impl() {     
  ...
```
这就是 Torque 编译器生成的内容，我们上面看到的是 `CodeStubAssemble` 类。

如果我们查看 `out/x64.release_gcc/gen/torque-generated/builtin-definitions-tq.h`，可以找到以下生成的行：
```c++
TFJ(MathIs42, 1, kReceiver, kX) \                                               
```
现在，有一个关于 [TF_BUILTIN](#tf_builtin) 宏的部分，它将创建函数声明、函数和类定义。

现在，在 `src/builtins/builtins.h` 中我们有以下宏：
```c++
class Builtins {
 public:

  enum Name : int32_t {
```
`SimpleInstallFunction` 的签名如下：
```c++
V8_NOINLINE Handle<JSFunction> SimpleInstallFunction(
    Isolate* isolate, Handle<JSObject> base, const char* name,
    Builtins::Name call, int len, bool adapt,
    PropertyAttributes attrs = DONT_ENUM) {
  Handle<String> internalized_name = isolate->factory()->InternalizeUtf8String(name);
  Handle<JSFunction> fun = SimpleCreateFunction(isolate, internalized_name, call, len, adapt);       
  JSObject::AddProperty(isolate, base, internalized_name, fun, attrs);          
  return fun;                                                                   
} 
```
所以我们看到该函数作为属性被添加到 Math 对象中。
注意我们还必须将 `kMathIs42` 添加到 `Builtins` 类中，它现在是我们上面提到的 `builtins_table_` 数组的一部分。

#### Transitioning/Transient
在 Torque 源文件中，我们有时可以看到声明为 `transient` 的类型，以及具有 `transitioning` 说明符的函数。在 V8 中，`HeapObjects` 可以在运行时更改（我认为一个例子是删除数组中的元素，这会将其转换为不同类型的数组，如 `HoleyElementArray` 等。TODO：验证并解释这一点）。而调用会导致此类转换的 JavaScript 函数会被标记为 `transitioning`。

#### Callables
类似于 JS/C++ 中的函数，但具有一些额外的能力，并且有几种不同类型的 Callables：

##### macro callables
这些对应于生成的 `CodeStubAssembler` C++ 代码，将在调用处内联。

##### builtin callables
这些将成为 V8 Builtins，其信息被添加到 `builtin-definitions.h` 中（通过包含 `torque-generated/builtin-definitions-tq.h`）。这只有一份副本，且将是函数调用，而不是像宏那样被内联。

##### runtime callables

##### intrinsic callables

#### 显式参数
Macros 和 Builtins 可以有参数。例如：
```
@export
macro HelloWorld1(msg: JSAny) {
  Print(msg);
}
```
我们可以从另一个宏中这样调用它：
```
@export
macro HelloWorld() {
  HelloWorld1('Hello World');
}
```

#### 隐式参数
在上一节中我们展示了显式参数，但我们也可以有隐式参数：
```
@export
macro HelloWorld2(implicit msg: JSAny)() {
  Print(msg);
}
@export
macro HelloWorld() {
  const msg = 'Hello implicit';
  HelloWorld2();
}
```

### 故障排除
包含 `src/objects/objects-inl.h` 时出现编译错误：
```console
/home/danielbevenius/work/google/v8_src/v8/src/objects/object-macros.h:263:14: error: no declaration matches ‘bool v8::internal::HeapObject::IsJSCollator() const’
```
这是否需要 i18n 支持？
```console
$ gn args --list out/x64.release_gcc | grep i18n
v8_enable_i18n_support
```

```console
usr/bin/ld: /tmp/ccJOrUMl.o: in function `v8::internal::MaybeHandle<v8::internal::Object>::Check() const':
/home/danielbevenius/work/google/v8_src/v8/src/handles/maybe-handles.h:44: undefined reference to `V8_Fatal(char const*, ...)'
collect2: error: ld returned 1 exit status
```
`V8_Fatal` 被引用但未在 `v8_monolith.a` 中定义：
```console
$ nm libv8_monolith.a | grep V8_Fatal | c++filt 
...
U V8_Fatal(char const*, int, char const*, ...)
```
我以为它可能定义在 `libv8_libbase.a` 中，但那里也是一样。
实际上，我找错了符号。这不来自 `logging.o` 对象文件。如果我们查看它，会发现：
```console
v8_libbase/logging.o:
...
0000000000000000 T V8_Fatal(char const*, int, char const*, ...)
```
在 `out/x64.release/obj/logging.o` 中我们可以找到它的定义：
```console
$ nm -C  libv8_libbase.a | grep -A 50 logging.o | grep V8_Fatal
0000000000000000 T V8_Fatal(char const*, int, char const*, ...)
```
`T` 表示该符号在 text 段中。
因此，如果链接器能够找到 `libv8_libbase.a`，它应该能够解析此符号。

所以我们需要确保链接器能找到库所在的目录（`-Wl,-Ldir`），并且会包含该库（`-Wl,-llibname`）。

设置好之后，我可以看到链接器可以打开存档：
```console
attempt to open /home/danielbevenius/work/google/v8_src/v8/out/x64.release_gcc/obj/libv8_libbase.so failed
attempt to open /home/danielbevenius/work/google/v8_src/v8/out/x64.release_gcc/obj/libv8_libbase.a succeeded
/home/danielbevenius/work/google/v8_src/v8/out/x64.release_gcc/obj/libv8_libbase.a
```
但我仍然遇到同样的链接错误。如果我们仔细看错误消息，可以看到是 `maybe-handles.h` 在报错。是否可能是链接时的顺序不正确？`libv8_libbase.a` 需要放在 `libv8_monolith` 之后。
我注意到的一点是，尽管找到了 `libv8_libbase.a` 库，但链接器似乎并没有实际读取对象文件。我看到它对 `libv8_monolith.a` 确实执行了此操作：
```console
(/home/danielbevenius/work/google/v8_src/v8/out/x64.release_gcc/obj/libv8_monolith.a)common-node-cache.o
```
嗯，实际上看函数的签名，它是 `V8_Fatal(char const*, ...)` 而不是 `char const*, int, char const*, ...)`。

对于 Debug 构建，它将是：
```c++
    void V8_Fatal(const char* file, int line, const char* format, ...);
```
否则：
```c++
    void V8_Fatal(const char* format, ...);
```
所以看起来我需要将 debug 设置为 false。这样，`logging.o` 中的 `V8_Fatal` 符号为：
```console
$ nm -C out/x64.release_gcc/obj/v8_libbase/logging.o | grep V8_Fatal
0000000000000000 T V8_Fatal(char const*, ...)
```

### V8 构建产物
当你指定 `v8_monolithic` 时，实际构建了什么：
选择此类型时，构建不能是组件构建（component build），对此有一个断言。在这种情况下，是一个静态库构建：
```
if (v8_monolithic) {                                                            
  # 组件构建不是单体构建。
  assert(!is_component_build)                                                   
                                                                                
  # 使用外部启动数据会产生单独的文件。
  assert(!v8_use_external_startup_data)                                         
  v8_static_library("v8_monolith") {                                            
    deps = [                                                                    
      ":v8",                                                                    
      ":v8_libbase",                                                            
      ":v8_libplatform",                                                        
      ":v8_libsampler",                                                         
      "//build/win:default_exe_manifest",                                       
    ]                                                                           
                                                                                
    configs = [ ":internal_config" ]                                            
  }                                                                             
}
```
注意内置函数被称为 `static_library`，因此它是一个可以在 `gni/v8.gni` 中找到的模板。

`v8_static_library`：
这将在编译时使用 `source_set` 而不是创建静态库。当设置为 false 时，对象文件将包含在链接器命令中。这可以加快构建速度，因为跳过了静态库的创建。但这对于像本项目这样从外部链接到 V8 并没有真正的帮助。

`is_component_build`：
这将把声明为组件的目标编译为共享库。`BUILD.gn` 中的所有 `v8_components` 都将在输出目录中构建为 `.so` 文件（而不是静态库所在的 `obj` 目录）。

因此，仅有的两个选项是 `v8_monolith` 或 `is_component_build`，后者的优势在于能够构建单个组件，而不必有时重新构建整个单体。

### wee8
可以生成 `libwee8`，这是一个仅支持 WebAssembly 而不支持 JavaScript 的库。
```shell
$ ninja -C out/wee8 wee8
```

### V8 内部 Isolate
`src/execution/isolate.h` 是你可以找到 `v8::internal::Isolate` 的地方。
```c++
class V8_EXPORT_PRIVATE Isolate final : private HiddenFactory {
```
`HiddenFactory` 只是为了允许 `Isolate` 私有继承自 `Factory`（可以在 `src/heap/factory.h` 中找到）。

### 启动流程详解
本节将通过使用本项目中的 `hello_world` 示例来介绍 V8 的启动过程：
```shell
$ LD_LIBRARY_PATH=../v8_src/v8/out/x64.release_gcc/ lldb ./hello-world
(lldb) br s -n main
Breakpoint 1: where = hello-world`main + 25 at hello-world.cc:41:38, address = 0x0000000000402821
```
```c++
    V8::InitializeExternalStartupData(argv[0]);
```
此调用将进入 `api.cc`，它只是将调用委托给内部（即 internal 命名空间）。如果你尝试步入此函数，你将直接跳到 `hello_world` 的下一行。这是因为我们编译 V8 时没有使用外部启动数据，所以此函数将为空：
```console
$ objdump -Cd out/x64.release_gcc/obj/v8_base_without_compiler/startup-data-util.o
Disassembly of section .text._ZN2v88internal37InitializeExternalStartupDataFromFileEPKc:

0000000000000000 <v8::internal::InitializeExternalStartupDataFromFile(char const*)>:
   0:	c3                   	retq
```
接下来，我们有：
```c++
    std::unique_ptr<Platform> platform = platform::NewDefaultPlatform();
```
这将进入 `src/libplatform/default-platform.cc`，它将创建一个新的 `DefaultPlatform`。

```c++
Isolate* isolate = Isolate::New(create_params);
```
这将调用 `Allocate`：
```c++
Isolate* isolate = Allocate();
```
```c++
Isolate* Isolate::Allocate() {
  return reinterpret_cast<Isolate*>(i::Isolate::New());
}
```

请记住，内部 `Isolate` 可以在 `src/execution/isolate.h` 中找到。在 `src/execution/isolate.cc` 中我们找到 `Isolate::New`：
```c++
Isolate* Isolate::New(IsolateAllocationMode mode) {
  std::unique_ptr<IsolateAllocator> isolate_allocator = std::make_unique<IsolateAllocator>(mode);
  void* isolate_ptr = isolate_allocator->isolate_memory();
  Isolate* isolate = new (isolate_ptr) Isolate(std::move(isolate_allocator));
```
所以我们首先创建一个 `IsolateAllocator` 实例，它将为单个 `Isolate` 实例分配内存。然后将其传递给 `Isolate` 构造函数，注意这里 `new` 的用法，这只是一个普通的堆分配。

默认的 `new` 运算符已被删除，并提供了一个接受 `void` 指针的重写，该指针直接返回：
```c++
  void* operator new(size_t, void* ptr) { return ptr; }
  void* operator new(size_t) = delete;
  void operator delete(void*) = delete;
```
在这种情况下，它只是返回由 `isolate_memory()` 分配的内存。这样做的原因是，使用 `new` 运算符不仅会调用 `new` 运算符，编译器还会添加一个对类型构造函数的调用，并传入分配内存的地址。
```c++
Isolate::Isolate(std::unique_ptr<i::IsolateAllocator> isolate_allocator)
    : isolate_data_(this),
      isolate_allocator_(std::move(isolate_allocator)),
      id_(isolate_counter.fetch_add(1, std::memory_order_relaxed)),
      allocator_(FLAG_trace_zone_stats
                     ? new VerboseAccountingAllocator(&heap_, 256 * KB)
                     : new AccountingAllocator()),
      builtins_(this),
      rail_mode_(PERFORMANCE_ANIMATION),
      code_event_dispatcher_(new CodeEventDispatcher()),
      jitless_(FLAG_jitless),
#if V8_SFI_HAS_UNIQUE_ID
      next_unique_sfi_id_(0),
#endif
      cancelable_task_manager_(new CancelableTaskManager()) {
```
注意 `isolate_data_` 将通过调用构造函数来填充，该构造函数接受一个指向 `Isolate` 的指针。
```c++
class IsolateData final {
 public:
  explicit IsolateData(Isolate* isolate) : stack_guard_(isolate) {}
```

回到 `Isolate` 的构造函数，我们有：
```c++
#define ISOLATE_INIT_LIST(V)                                                   \
  /* Assembler state. */                                                       \
  V(FatalErrorCallback, exception_behavior, nullptr)                           \
  ...

#define ISOLATE_INIT_EXECUTE(type, name, initial_value) \                           
  name##_ = (initial_value);                                                        
  ISOLATE_INIT_LIST(ISOLATE_INIT_EXECUTE)                                           
#undef ISOLATE_INIT_EXECUTE
```
让我们展开第一个条目以了解发生了什么：
```c++
   exception_behavior_ = (nullptr);
   oom_behavior_ = (nullptr);
   event_logger_ = (nullptr);
   allow_code_gen_callback_ = (nullptr);
   modify_code_gen_callback_ = (nullptr);
   allow_wasm_code_gen_callback_ = (nullptr);
   wasm_module_callback_ = (&NoExtension);
   wasm_instance_callback_ = (&NoExtension);
   wasm_streaming_callback_ = (nullptr);
   wasm_threads_enabled_callback_ = (nullptr);
   wasm_load_source_map_callback_ = (nullptr);
   relocatable_top_ = (nullptr);
   string_stream_debug_object_cache_ = (nullptr);
   string_stream_current_security_token_ = (Object());
   api_external_references_ = (nullptr);
   external_reference_map_ = (nullptr);
   root_index_map_ = (nullptr);
   default_microtask_queue_ = (nullptr);
   turbo_statistics_ = (nullptr);
   code_tracer_ = (nullptr);
   per_isolate_assert_data_ = (0xFFFFFFFFu);
   promise_reject_callback_ = (nullptr);
   snapshot_blob_ = (nullptr);
   code_and_metadata_size_ = (0);
   bytecode_and_metadata_size_ = (0);
   external_script_source_size_ = (0);
   is_profiling_ = (false);
   num_cpu_profilers_ = (0);
   formatting_stack_trace_ = (false);
   debug_execution_mode_ = (DebugInfo::kBreakpoints);
   code_coverage_mode_ = (debug::CoverageMode::kBestEffort);
   type_profile_mode_ = (debug::TypeProfileMode::kNone);
   last_stack_frame_info_id_ = (0);
   last_console_context_id_ = (0);
   inspector_ = (nullptr);
   next_v8_call_is_safe_for_termination_ = (false);
   only_terminate_in_safe_scope_ = (false);
   detailed_source_positions_for_profiling_ = (FLAG_detailed_line_info);
   embedder_wrapper_type_index_ = (-1);
   embedder_wrapper_object_index_ = (-1);
```
因此，在预处理器完成后，此列表中的所有条目都将成为 `Isolate` 类的私有成员。还将有公共访问器来获取和设置这些初始值（这是上面 `ISOLATE_INIT_LIST` 中的最后一个条目）。

回到 `isolate.cc` 构造函数，我们有：
```c++
#define ISOLATE_INIT_ARRAY_EXECUTE(type, name, length) \
  memset(name##_, 0, sizeof(type) * length);
  ISOLATE_INIT_ARRAY_LIST(ISOLATE_INIT_ARRAY_EXECUTE)
#undef ISOLATE_INIT_ARRAY_EXECUTE
```
```c++
  v8::internal::TorqueGeneratedOddball<v8::internal::Oddball, v8::internal::PrimitiveHeapObject> = {
    v8::internal::PrimitiveHeapObject = {
      v8::internal::TorqueGeneratedPrimitiveHeapObject<v8::internal::PrimitiveHeapObject, v8::internal::HeapObject> = {
        v8::internal::HeapObject = {
          v8::internal::Object = {
            v8::internal::TaggedImpl<v8::internal::HeapObjectReferenceType::STRONG, unsigned long> = (ptr_ = 0x00000f750804030d)
          }
        }
      }
    }
  }
}
```
你也可以直接将其转换为对象并尝试打印：
```console
(lldb) expr -A -f hex  -- v8::internal::Object(isolate_->isolate_data_.roots_.roots_[4]).Print()
#undefined
```
这实际上是 `Oddball UndefinedValue`，所以在这里是有意义的。
有了 `roots_` 数组中的这个值，我们可以使用函数 `ReadOnlyRoots::undefined_value()`：
```console
(lldb) expr v8::internal::ReadOnlyRoots(&isolate_->heap_).undefined_value()
(v8::internal::Oddball) $265 = {
  v8::internal::TorqueGeneratedOddball<v8::internal::Oddball, v8::internal::PrimitiveHeapObject> = {
    v8::internal::PrimitiveHeapObject = {
      v8::internal::TorqueGeneratedPrimitiveHeapObject<v8::internal::PrimitiveHeapObject, v8::internal::HeapObject> = {
        v8::internal::HeapObject = {
          v8::internal::Object = {
            v8::internal::TaggedImpl<v8::internal::HeapObjectReferenceType::STRONG, unsigned long> = (ptr_ = 16995320070925)
          }
        }
      }
    }
  }
}
```
那么这些 Roots 是如何使用的，以 `undefined_value` 为例？
大多数（也许是全部）需要的东西都通过 `Factory` 进行，内部 `Isolate` 就是其一种类型。在 Factory 中我们可以找到：
```c++
Handle<Oddball> Factory::undefined_value() {
  return Handle<Oddball>(&isolate()->roots_table()[RootIndex::kUndefinedValue]);
}
```
注意这基本上就是我们之前在调试器中所做的，但这里它被包装在 `Handle` 中，以便 GC 可以跟踪它。

单元测试 [isolate_test](./test/isolate_test.cc) 探索了内部 Isolate，并有上述方法的用法示例。

`InitwithSnapshot` 将调用 `Isolate::Init`：
```c++
bool Isolate::Init(ReadOnlyDeserializer* read_only_deserializer,
                   StartupDeserializer* startup_deserializer) {

#define ASSIGN_ELEMENT(CamelName, hacker_name)                  \
  isolate_addresses_[IsolateAddressId::k##CamelName##Address] = \
      reinterpret_cast<Address>(hacker_name##_address());
  FOR_EACH_ISOLATE_ADDRESS_NAME(ASSIGN_ELEMENT)
#undef ASSIGN_ELEMENT
```
```c++
  Address isolate_addresses_[kIsolateAddressCount + 1] = {};
```
```console
(gdb) p isolate_addresses_
$16 = {0 <repeats 13 times>}
```

让我们看看 `Isolate::Init` 中展开的代码：
```console
$ clang++ -I./out/x64.release/gen -I. -I./include -E src/execution/isolate.cc > output
```
```c++
isolate_addresses_[IsolateAddressId::kHandlerAddress] = reinterpret_cast<Address>(handler_address());
isolate_addresses_[IsolateAddressId::kCEntryFPAddress] = reinterpret_cast<Address>(c_entry_fp_address());
isolate_addresses_[IsolateAddressId::kCFunctionAddress] = reinterpret_cast<Address>(c_function_address());
isolate_addresses_[IsolateAddressId::kContextAddress] = reinterpret_cast<Address>(context_address());
isolate_addresses_[IsolateAddressId::kPendingExceptionAddress] = reinterpret_cast<Address>(pending_exception_address());
isolate_addresses_[IsolateAddressId::kPendingHandlerContextAddress] = reinterpret_cast<Address>(pending_handler_context_address());
isolate_addresses_[IsolateAddressId::kPendingHandlerEntrypointAddress] = reinterpret_cast<Address>(pending_handler_entrypoint_address());
isolate_addresses_[IsolateAddressId::kPendingHandlerConstantPoolAddress] = reinterpret_cast<Address>(pending_handler_constant_pool_address());
isolate_addresses_[IsolateAddressId::kPendingHandlerFPAddress] = reinterpret_cast<Address>(pending_handler_fp_address());
isolate_addresses_[IsolateAddressId::kPendingHandlerSPAddress] = reinterpret_cast<Address>(pending_handler_sp_address());
isolate_addresses_[IsolateAddressId::kExternalCaughtExceptionAddress] = reinterpret_cast<Address>(external_caught_exception_address());
isolate_addresses_[IsolateAddressId::kJSEntrySPAddress] = reinterpret_cast<Address>(js_entry_sp_address());
```
然后函数（如 `handler_address()`）实现为：
```c++ 
inline Address* handler_address() { return &thread_local_top()->handler_; }   
```
```console
(gdb) x/x isolate_addresses_[0]
0x1a3500003240:	0x00000000
```
在程序的这一点上，我们只是将条目设置为指向 `ThreadLocalTop` 中指定的地址。在初始化时，它们大多会被初始化为 `kNullAddress`：
```c++
static const Address kNullAddress = 0;
```
注意上面的函数返回指针，所以稍后这些指针可以更新为指向某些内容。这在什么时候/如何发生？让我们继续寻找答案……

回到 `Isolate::Init`，我们有：
```c++
  compilation_cache_ = new CompilationCache(this);
  descriptor_lookup_cache_ = new DescriptorLookupCache();
  inner_pointer_to_code_cache_ = new InnerPointerToCodeCache(this);
  global_handles_ = new GlobalHandles(this);
  eternal_handles_ = new EternalHandles();
  bootstrapper_ = new Bootstrapper(this);
  handle_scope_implementer_ = new HandleScopeImplementer(this);
  load_stub_cache_ = new StubCache(this);
  store_stub_cache_ = new StubCache(this);
  materialized_object_store_ = new MaterializedObjectStore(this);
  regexp_stack_ = new RegExpStack();
  regexp_stack_->isolate_ = this;
  date_cache_ = new DateCache();
  heap_profiler_ = new HeapProfiler(heap());
  interpreter_ = new interpreter::Interpreter(this);

  compiler_dispatcher_ =
      new CompilerDispatcher(this, V8::GetCurrentPlatform(), FLAG_stack_size);

  // 设置对象堆。
  DCHECK(!heap_.HasBeenSetUp());
  heap_.SetUp();

  ...
  InitializeThreadLocal();
```
让我们看看 `InitializeThreadLocal`：
```c++
void Isolate::InitializeThreadLocal() {
  thread_local_top()->Initialize(this);
  clear_pending_exception();
  clear_pending_message();
  clear_scheduled_exception();
}
```
```c++
void Isolate::clear_pending_exception() {
  DCHECK(!thread_local_top()->pending_exception_.IsException(this));
  thread_local_top()->pending_exception_ = ReadOnlyRoots(this).the_hole_value();
}
```
`ReadOnlyRoots`：
```c++
#define ROOT_ACCESSOR(Type, name, CamelName) \
  V8_INLINE class Type name() const;         \
  V8_INLINE Handle<Type> name##_handle() const;

  READ_ONLY_ROOT_LIST(ROOT_ACCESSOR)
#undef ROOT_ACCESSOR
```
这将展开为一系列函数声明，如下所示：
```console
$ clang++ -I./out/x64.release/gen -I. -I./include -E src/roots/roots.h > output
```
```c++
inline __attribute__((always_inline)) class Map free_space_map() const;
inline __attribute__((always_inline)) Handle<Map> free_space_map_handle() const;
```
`Map` 类是所有 `HeapObject` 用来描述其结构的。注意还声明了一个 `Handle<Map>`。
这些是由 `roots-inl.h` 中的宏生成的：
```c++
Map ReadOnlyRoots::free_space_map() const { 
  ((void) 0);
  return Map::unchecked_cast(Object(at(RootIndex::kFreeSpaceMap)));
} 

Handle<Map> ReadOnlyRoots::free_space_map_handle() const {
  ((void) 0);
  return Handle<Map>(&at(RootIndex::kFreeSpaceMap));
}
```
注意这使用了前面提到的 `RootIndex` 枚举：
```c++
  return Map::unchecked_cast(Object(at(RootIndex::kFreeSpaceMap)));
```
在 `object/map.h` 中有以下行：
```c++
  DECL_CAST(Map)
```
可以在 `objects/object-macros.h` 中找到：
```c++
#define DECL_CAST(Type)                                 \
  V8_INLINE static Type cast(Object object);            \
  V8_INLINE static Type unchecked_cast(Object object) { \
    return bit_cast<Type>(object);                      \
  }
```
这将展开为类似：
```c++
  static Map cast(Object object);
  static Map unchecked_cast(Object object) {
    return bit_cast<Map>(object);
  }
```
`Object` 部分是接受 `Address` 的 `Object` 构造函数：
```c++
  explicit constexpr Object(Address ptr) : TaggedImpl(ptr) {}
```
剩下的是 `at` 函数，它是 `ReadOnlyRoots` 中的一个私有函数：
```c++
  V8_INLINE Address& at(RootIndex root_index) const;
```

现在我们回到 `Isolate::Init`，在调用 `InitializeThreadLocal` 之后，我们有：
```c++
setup_delegate_->SetupBuiltins(this);
```

在 `api.cc` 的下一行中，`i::OBJECT_TEMPLATE_INFO_TYPE` 是从哪里来的：
```c++
  i::Handle<i::Struct> struct_obj = isolate->factory()->NewStruct(
      i::OBJECT_TEMPLATE_INFO_TYPE, i::AllocationType::kOld);
```

### InstanceType
枚举 `InstanceType` 定义在 `src/objects/instance-type.h` 中：
```c++
#include "torque-generated/instance-types-tq.h" 

enum InstanceType : uint16_t {
  ...   
#define MAKE_TORQUE_INSTANCE_TYPE(TYPE, value) TYPE = value,                    
  TORQUE_ASSIGNED_INSTANCE_TYPES(MAKE_TORQUE_INSTANCE_TYPE)                     
#undef MAKE_TORQUE_INSTANCE_TYPE 
  ...
};
```
在 `gen/torque-generated/instance-types-tq.h` 中我们可以找到：
```c++
#define TORQUE_ASSIGNED_INSTANCE_TYPES(V) \                                     
  ...
  V(OBJECT_TEMPLATE_INFO_TYPE, 79) \                                      
  ...
```
在 `src/objects/objects-definitions.h` 中有一个列表：
```c++
#define STRUCT_LIST_GENERATOR_BASE(V, _)                                      \
  ...
  V(_, OBJECT_TEMPLATE_INFO_TYPE, ObjectTemplateInfo, object_template_info)   \
  ...
```
```c++
template <typename Impl>
Handle<Struct> FactoryBase<Impl>::NewStruct(InstanceType type,
                                            AllocationType allocation) {
  Map map = Map::GetInstanceTypeMap(read_only_roots(), type);
```
如果我们查看 `map.cc` 中的 `Map::GetInstanceTypeMap`，会发现：
```c++
  Map map;
  switch (type) {
#define MAKE_CASE(TYPE, Name, name) \
  case TYPE:                        \
    map = roots.name##_map();       \
    break;
    STRUCT_LIST(MAKE_CASE)
#undef MAKE_CASE
```
现在，我们知道我们的类型是：
```console
(gdb) p type
$1 = v8::internal::OBJECT_TEMPLATE_INFO_TYPE
```
```c++
    map = roots.object_template_info_map();       \
```
我们可以检查 `roots.cc` 的预处理器输出，发现：
```c++
Map ReadOnlyRoots::object_template_info_map() const { 
  ((void) 0);
  return Map::unchecked_cast(Object(at(RootIndex::kObjectTemplateInfoMap)));
}
```
这是我们之前见过的。

我遇到的一件事是想使用 `objects.cc` 中为 `InstanceType` 定义的重载 `<<` 运算符来打印 `InstanceType`。
```c++
std::ostream& operator<<(std::ostream& os, InstanceType instance_type) {
  switch (instance_type) {
#define WRITE_TYPE(TYPE) \
  case TYPE:             \
    return os << #TYPE;
    INSTANCE_TYPE_LIST(WRITE_TYPE)
#undef WRITE_TYPE
  }
  UNREACHABLE();
}
```
我使用的代码如下：
```c++
  i::InstanceType type = map.instance_type();
  std::cout << "object_template_info_map type: " << type << '\n';
```
这将导致调用 `UNREACHABLE()` 函数并抛出致命错误。但请注意，以下行可以工作：
```c++
  std::cout << "object_template_info_map type: " << v8::internal::OBJECT_TEMPLATE_INFO_TYPE << '\n';
```
并打印：
```console
object_template_info_map type: OBJECT_TEMPLATE_INFO_TYPE
```
在上面的 switch/case 块中，此值的 case 是：
```c++
  case OBJECT_TEMPLATE_INFO_TYPE:
    return os << "OBJECT_TEMPLATE_INFO_TYPE"
```
当调用 `map.instance_type()` 时，它返回一个值 `1023`，但 `OBJECT_TEMPLATE_INFO_TYPE` 的值是：
```c++
OBJECT_TEMPLATE_INFO_TYPE = 79
```
我们可以使用以下代码确认这一点：
```console
  std::cout << "object_template_info_map type: " << static_cast<uint16_t>(v8::internal::OBJECT_TEMPLATE_INFO_TYPE) << '\n';
```
这将打印：
```console
object_template_info_map type: 79
```

### IsolateData

### Context 创建
当我们使用以下代码创建一个新 Context 时：
```c++
  Local<ObjectTemplate> global = ObjectTemplate::New(isolate_);
  Local<Context> context = Context::New(isolate_, nullptr, global);
```
`include/v8.h` 中的 `Context` 类声明 `New` 如下：
```c++
static Local<Context> New(Isolate* isolate,
    ExtensionConfiguration* extensions = nullptr,
    MaybeLocal<ObjectTemplate> global_template = MaybeLocal<ObjectTemplate>(),
    MaybeLocal<Value> global_object = MaybeLocal<Value>(),
    DeserializeInternalFieldsCallback internal_fields_deserializer = DeserializeInternalFieldsCallback(),
    MicrotaskQueue* microtask_queue = nullptr);
```

当步入 `Context::New(isolate_, nullptr, global)` 时，首先会在 `v8.h` 中 `DeserializeInternalFieldsCallback` 的构造函数中中断，该构造函数具有回调函数和 `data_args` 的默认值（均为 `nullptr`）。之后 gdb 会在 `MaybeLocal<Value>` 中中断，并将 `val_` 设置为 `nullptr`。接下来它会在 `global` 值的 `Local::operator*` 中中断，然后该值被传递给 `MaybeLocal<v8::ObjectTemplate>` 构造函数。在这些断点之后，断点将位于 `api.cc` 和 `v8::Context::New`。`New` 将调用 `api.cc` 中的 `NewContext`。

会有一些检查和日志记录/追踪，然后调用 `CreateEnvironment`：
```c++
i::Handle<i::Context> env = CreateEnvironment<i::Context>(                         
    isolate,
    extensions,
    global_template, 
    global_object,                           
    context_snapshot_index, 
    embedder_fields_deserializer, 
    microtask_queue); 
```
`CreateEnvironment` 的第一行是：
```c++
ENTER_V8_FOR_NEW_CONTEXT(isolate);
```
这是定义在 `api.cc` 中的一个宏：
```c++
i::VMState<v8::OTHER> __state__((isolate)); \                                 
i::DisallowExceptions __no_exceptions__((isolate)) 
```
所以我们中断的第一个断点将是 `execution/vm-state-inl.h` 和 `VMState` 的构造函数：
```c++
template <StateTag Tag>                                                         
VMState<Tag>::VMState(Isolate* isolate)                                         
    : isolate_(isolate), previous_tag_(isolate->current_vm_state()) {           
  isolate_->set_current_vm_state(Tag);                                          
} 
```
在 gdb 中你会看到：
```console
(gdb) s
v8::internal::VMState<(v8::StateTag)5>::VMState (isolate=0x372500000000, this=<synthetic pointer>) at ../../src/api/api.cc:6005
6005	      context_snapshot_index, embedder_fields_deserializer, microtask_queue);
(gdb) s
v8::internal::Isolate::current_vm_state (this=0x372500000000) at ../../src/execution/isolate.h:1072
1072	  THREAD_LOCAL_TOP_ACCESSOR(StateTag, current_vm_state)
```
获取它们：
```c++
  inline Handle<Object> receiver() const;                                       
  inline Handle<JSFunction> target() const;                                     
  inline Handle<HeapObject> new_target() const;
```

### NativeContext
可以在 `src/objects/contexts.h` 中找到，定义如下：
```c++
class NativeContext : public Context {
 public:

  DECL_PRIMITIVE_ACCESSORS(microtask_queue, MicrotaskQueue*)

  V8_EXPORT_PRIVATE void AddOptimizedCode(Code code);
  void SetOptimizedCodeListHead(Object head);
  Object OptimizedCodeListHead();
  void SetDeoptimizedCodeListHead(Object head);
  Object DeoptimizedCodeListHead();
  inline OSROptimizedCodeCache GetOSROptimizedCodeCache();
  void ResetErrorsThrown();
  void IncrementErrorsThrown();
  int GetErrorsThrown();
```

在 `src/parsing/parser.h` 中我们可以找到：
```c++
class V8_EXPORT_PRIVATE Parser : public NON_EXPORTED_BASE(ParserBase<Parser>) { 
  ...
  enum CompletionKind {                                                             
    kNormalCompletion,                                                              
    kThrowCompletion,                                                               
    kAbruptCompletion                                                               
  };
```
但我找不到这个枚举的任何用法？

#### 内部字段/方法
当你看到类似 `[[Notation]]` 的内容时，可以将其视为对象中的一个字段，它不暴露给 JavaScript 用户代码，而是 JavaScript 引擎内部使用的。这些也可以用于内部方法。
