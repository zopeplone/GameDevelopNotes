### Promises
Promise 是一个具有状态的对象。例如，我们可以使用以下调用创建 promise：

```cpp
  i::Handle<i::JSPromise> promise = factory->NewJSPromise();
  print_handle(promise);
```
这将输出：
```console
0x3e46080c1cb9: [JSPromise]
 - map: 0x3e46081c0fa9 <Map(HOLEY_ELEMENTS)> [FastProperties]
 - prototype: 0x3e46082063ed <Object map = 0x3e46081c0fd1>
 - elements: 0x3e46080406e9 <FixedArray[0]> [HOLEY_ELEMENTS]
 - status: pending
 - reactions: 0
 - has_handler: 0
 - handled_hint: 0
 - properties: 0x3e46080406e9 <FixedArray[0]> {}
```
`JSPromise` 的实现可以在 `src/objects/js-promise.h` 中找到。

现在，`factory->NewJSPromise` 如下所示：
```cpp
Handle<JSPromise> Factory::NewJSPromise() {
  Handle<JSPromise> promise = NewJSPromiseWithoutHook();
  isolate()->RunPromiseHook(PromiseHookType::kInit, promise, undefined_value());
  return promise;
}
Handle<JSPromise> Factory::NewJSPromiseWithoutHook() {
  Handle<JSPromise> promise =
      Handle<JSPromise>::cast(NewJSObject(isolate()->promise_function()));
  promise->set_reactions_or_result(Smi::zero());
  promise->set_flags(0);
  ZeroEmbedderFields(promise);
  DCHECK_EQ(promise->GetEmbedderFieldCount(), v8::Promise::kEmbedderFieldCount);
  return promise;
}
```
注意 `isolate->promise_function()` 它返回什么，在哪里定义：
```console
$ gdb ./test/promise_test
(gdb) b Factory::NewJSPromise
(gdb) r
(gdb) p isolate()->promise_function()
$1 = {<v8::internal::HandleBase> = {location_ = 0x20a7c68}, <No data fields>}
```
这并没有告诉我太多。但如果我们查看 `src/execution/isolate-inl.h`，我们可以找到：
```cpp
#define NATIVE_CONTEXT_FIELD_ACCESSOR(index, type, name)    \
  Handle<type> Isolate::name() {                            \
    return Handle<type>(raw_native_context().name(), this); \
  }                                                         \
  bool Isolate::is_##name(type value) {                     \
    return raw_native_context().is_##name(value);           \
  }
NATIVE_CONTEXT_FIELDS(NATIVE_CONTEXT_FIELD_ACCESSOR)
#undef NATIVE_CONTEXT_FIELD_ACCESSOR
```
而 `NATIVE_CONTEXT_FIELDS` 可以在 `src/objects/contexts.h` 中找到，该文件由 `isolate.h` 包含（而 `isolate-inl.h` 又包含 `isolate.h`）。在 `contexts.h` 中我们发现：
```cpp
#define NATIVE_CONTEXT_FIELDS(V)
  ...
  V(PROMISE_FUNCTION_INDEX, JSFunction, promise_function)
```
因此，预处理器将为 promise_function 生成以下函数：
```cpp
inline Handle<JSFunction> promise_function();
inline bool is_promise_function(JSFunction value);

Handle<JSFunction> Isolate::promise_function() {
  return Handle<JSFunction>(raw_native_context().promise_function(), this);
}
bool Isolate::is_promise_function(JSFunction value) {
  return raw_native_context().is_promise_function(value);
}
```
以及从上面调用的函数（在 `src/objects/contexts.h` 中）：
```cpp
inline void set_promise_function(JSFunction value);
inline bool is_promise_function(JSFunction value) const;
inline JSFunction promise_function() const;

void Context::set_promise_function(JSFunction value) {
  ((void) 0);
  set(PROMISE_FUNCTION_INDEX, value);
}
bool Context::is_promise_function(JSFunction value) const {
  ((void) 0);
  return JSFunction::cast(get(PROMISE_FUNCTION_INDEX)) == value;
}
JSFunction Context::promise_function() const {
  ((void) 0);
  return JSFunction::cast(get(PROMISE_FUNCTION_INDEX));
}
```
这就回答了函数的声明和定义位置以及返回内容的问题。

我们可以在 `src/builtins/promise-constructor.tq` 中找到 Torque 源文件，其中有引用 ECMAScript 规范的注释。在我们的例子中，这是 [promise-executor](https://tc39.es/ecma262/#sec-promise-executor)：
```cpp
  transitioning javascript builtin                                                 
  PromiseConstructor(                                                              
      js-implicit context: NativeContext, receiver: JSAny,                         
      newTarget: JSAny)(executor: JSAny): JSAny {
   // 1. 如果 NewTarget 是 undefined，抛出 TypeError 异常。
   if (newTarget == Undefined) {
     ThrowTypeError(MessageTemplate::kNotAPromise, newTarget);
   }
```
对于生成的 C++ 代码，我们可以在 `out/x64.release_gcc/gen/torque-generated/src/builtins/promise-constructor-tq-csa.cc` 中查看。

现在，如果我们查看规范和 Torque 源码，我们可以发现规范中的第一步是：
```
1. If NewTarget is undefined, throw a TypeError exception.
```
而在 Torque 源文件中：
```
  if (newTarget == Undefined) {
      ThrowTypeError(MessageTemplate::kNotAPromise, newTarget);
  }
```
以及为此生成的 CodeStubAssembler C++ 源码：
```cpp
TF_BUILTIN(PromiseConstructor, CodeStubAssembler) {                             
  compiler::CodeAssemblerState* state_ = state();
  compiler::CodeAssembler ca_(state());
  TNode<Object> parameter2 = UncheckedCast<Object>(Parameter(Descriptor::kJSNewTarget));
  ...

  TNode<Oddball> tmp0;
  TNode<BoolT> tmp1;
  if (block0.is_used()) {
    ca_.Bind(&block0);
    ca_.SetSourcePosition("../../src/builtins/promise-constructor.tq", 51);
    tmp0 = Undefined_0(state_);
    tmp1 = CodeStubAssembler(state_).TaggedEqual(TNode<Object>{parameter2}, TNode<HeapObject>{tmp0});
    ca_.Branch(tmp1, &block1, std::vector<Node*>{}, &block2, std::vector<Node*>{});
  }             

  if (block1.is_used()) {
    ca_.Bind(&block1);
    ca_.SetSourcePosition("../../src/builtins/promise-constructor.tq", 52);
    CodeStubAssembler(state_).ThrowTypeError(TNode<Context>{parameter0}, MessageTemplate::kNotAPromise, TNode<Object>{parameter2});
  }
```
现在，`TF_BUILTIN` 是 `src/builtins/builtins-utils-gen.h` 中的宏：
```cpp
#define TF_BUILTIN(Name, AssemblerBase)                                 \
  class Name##Assembler : public AssemblerBase {                        \
   public:                                                              \
    using Descriptor = Builtin_##Name##_InterfaceDescriptor;            \
                                                                        \
    explicit Name##Assembler(compiler::CodeAssemblerState* state)       \
        : AssemblerBase(state) {}                                       \
    void Generate##Name##Impl();                                        \
                                                                        \
    Node* Parameter(Descriptor::ParameterIndices index) {               \
      return CodeAssembler::Parameter(static_cast<int>(index));         \
    }                                                                   \
  };                                                                    \
  void Builtins::Generate_##Name(compiler::CodeAssemblerState* state) { \
    Name##Assembler assembler(state);                                   \
    state->SetInitialDebugInformation(#Name, __FILE__, __LINE__);       \
    if (Builtins::KindOf(Builtins::k##Name) == Builtins::TFJ) {         \
      assembler.PerformStackCheck(assembler.GetJSContextParameter());   \
    }                                                                   \
    assembler.Generate##Name##Impl();                                   \
  }                                                                     \
  void Name##Assembler::Generate##Name##Impl()
```

因此上述内容将被预处理器展开为：
```cpp
TF_BUILTIN(PromiseConstructor, CodeStubAssembler) {                             
class PromiseConstructorAssembler : public CodeStubAssembler {
 public:
  using Descriptor = Builtin_PromiseConstructor_InterfaceDescriptor;

  explicit PromiseConstructorAssembler(compiler::CodeAssemblerState* state) :
      CodeStubAssembler(state) {}
  void GeneratePromiseConstructorImpl;

  Node* Parameter(Descriptor::ParameterIndices index) {
    return CodeAssembler::Parameter(static_cast<int>(index));
  }
};

void Builtins::Generate_PromiseConstructor(compiler::CodeAssemblerState* state) {
  PromiseConstructorAssembler assembler(state);
  state->SetInitialDebugInformation(PromiseConstructor, __FILE__, __LINE__);
  if (Builtins::KindOf(Builtins::kPromiseConstructor) == Builtins::TFJ) {
    assembler.PerformStackCheck(assembler.GetJSContextParameter());
  }
```
