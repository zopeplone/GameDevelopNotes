### Exception
本文档包含有关 V8 中 Exception/Error 处理的笔记。

### Exceptions
当调用一个函数时，异常是如何在调用链中报告/触发的？

```c++
  Local<FunctionTemplate> ft = FunctionTemplate::New(isolate_, function_callback);
  Local<Function> function = ft->GetFunction(context).ToLocalChecked();
  Local<Object> recv = Object::New(isolate_);
  MaybeLocal<Value> ret = function->Call(context, recv, 0, nullptr);
```
`function->Call` 最终会进入 `src/api/api.cc` 中的 `Function::Call`，然后调用 `v8::internal::Execution::Call`：
```c++
  has_pending_exception = !ToLocal<Value>(                                           
      i::Execution::Call(isolate, self, recv_obj, argc, args), &result);             
  RETURN_ON_FAILED_EXECUTION(Value);                                                 
  RETURN_ESCAPED(result);            
```
注意，`Call` 的结果是 `MaybeHandle<Object>`，它将被传递给 `ToLocal<Value>`，定义在 `api.h` 中：
```c++
template <class T>                                                              
inline bool ToLocal(v8::internal::MaybeHandle<v8::internal::Object> maybe,      
                    Local<T>* local) {                                          
  v8::internal::Handle<v8::internal::Object> handle;                            
  if (maybe.ToHandle(&handle)) {                                                   
    *local = Utils::Convert<v8::internal::Object, T>(handle);                   
    return true;                                                                
  }                                                                                
  return false;                                                                 
```

Execution::Call 将调用 `Invoke`：
```c++
V8_WARN_UNUSED_RESULT MaybeHandle<Object> Invoke(Isolate* isolate,              
                                                 const InvokeParams& params) {
    ...
    auto value = Builtins::InvokeApiFunction(                                 
          isolate, params.is_construct, function, receiver, params.argc,        
          params.argv, Handle<HeapObject>::cast(params.new_target));            

    bool has_exception = value.is_null();                                     
    DCHECK(has_exception == isolate->has_pending_exception());                
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

```c++
  VMState<EXTERNAL> state(isolate);                                             
  ExternalCallbackScope call_scope(isolate, FUNCTION_ADDR(f));                  
  FunctionCallbackInfo<v8::Value> info(values_, argv_, argc_); 
  f(info);                                                                      
  return GetReturnValue<Object>(isolate);                                       
}
```
`f(info)` 是指向我们的回调函数的函数指针：
```console
(lldb) expr f
(v8::FunctionCallback) $0 = 0x0000000000415fa6 (exceptions_test`function_callback(v8::FunctionCallbackInfo<v8::Value> const&) at exceptions_test.cc:16:65)
```
当我们的回调调用 `info.GetReturnValue()` 时，它从 args 中获取 ReturnType 槽位，我们将它的值设置为 8，然后返回。
接下来，调用 `GetReturnValue`，然后返回到 `HandleApiCallHelper`，其中包含以下宏：
```c++
  RETURN_EXCEPTION_IF_SCHEDULED_EXCEPTION(isolate, Object);
```
该宏将被预处理器展开为：
```c++
  do {
    Isolate* __isolate__ = (isolate);
    ((void) 0);
    if (__isolate__->has_scheduled_exception()) {
      __isolate__->PromoteScheduledException();
      return MaybeHandle<Object>();
    }
  } while (false);
```
因此在这种情况下，`has_scheduled_exception` 将为 true，请注意，一个空的 MaybeHandle 将被返回给 `Invoke`。
```c++
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
    }                                    
```
这将返回到 `v8::Function::Call`：
```c++
  has_pending_exception = !ToLocal<Value>(                                           
      i::Execution::Call(isolate, self, recv_obj, argc, args), &result);             
  RETURN_ON_FAILED_EXECUTION(Value);                                                 
  RETURN_ESCAPED(result);            
```
我们可以使用以下命令展开这些宏：
```console
$ g++ -Iout/learning_v8/gen/ -I./include -I. -E src/api/api.cc
```
```c++
  do {
    if (has_pending_exception) {
      call_depth_scope.Escape();
      return MaybeLocal<Value>(); }
  } while (false);
  return handle_scope.Escape(result);;
```

### 抛出异常
当调用一个 Function 时，可以使用以下方式抛出异常：
```c++
  isolate->ThrowException(String::NewFromUtf8(isolate, "some error").ToLocalChecked());
```
`ThrowException` 可以在 `src/api/api.cc` 中找到，它的作用是：
```c++
  ENTER_V8_DO_NOT_USE(isolate);                                                 
  // 如果我们传入一个空 handle，则抛出一个 undefined 异常，
  // 以便更优雅地处理内存不足的情况。
  if (value.IsEmpty()) {                                                        
    isolate->ScheduleThrow(i::ReadOnlyRoots(isolate).undefined_value());             
  } else {                                                                           
    isolate->ScheduleThrow(*Utils::OpenHandle(*value));                               
  }                                                                                  
  return v8::Undefined(reinterpret_cast<v8::Isolate*>(isolate));      
```
让我们仔细看看 `ScheduleThrow`：
```c++
void Isolate::ScheduleThrow(Object exception) {                                     
  Throw(exception);                                                                 
  PropagatePendingExceptionToExternalTryCatch();                                     
  if (has_pending_exception()) {                                                    
    thread_local_top()->scheduled_exception_ = pending_exception();                 
    thread_local_top()->external_caught_exception_ = false;                         
    clear_pending_exception();                                                      
  }                                                                                 
}
```
`Throw` 最终将 pending_exception 设置为传入的异常。
接下来，将调用 `PropagatePendingExceptionToExternalTryCatch`。这就是 `TryCatch` handler 发挥作用的地方。如果已经注册了一个 TryCatch（在撰写本文时我还没有注册，但会添加一个来尝试并验证）。
此部分的代码如下：
```c++
  v8::TryCatch* handler = try_catch_handler();
  handler->can_continue_ = true;
  handler->has_terminated_ = false;
  handler->exception_ = reinterpret_cast<void*>(pending_exception().ptr());
  // 仅当我们获得实际消息时才传播到外部 try-catch。
  if (thread_local_top()->pending_message_obj_.IsTheHole(this)) return true;
  handler->message_obj_ = reinterpret_cast<void*>(thread_local_top()->pending_message_obj_.ptr());
```  
当 TryCatch 被创建时，其构造函数将调用 `RegisterTryCatchHandler`，该函数将设置 thread_local_top 的 try_catch_handler，该 handler 通过调用 `try_catch_handler()` 在上面被获取。

在此之前，会调用 `IsJavaScriptHandlerOnTop`：
```c++
// 对于不可捕获的异常，JavaScript handler 不能在顶部。
if (!is_catchable_by_javascript(exception)) return false;
```
```c++
  return exception != ReadOnlyRoots(heap()).termination_exception();               
}
```
TODO: 我真的需要更好地理解这一点，以及捕获/处理异常的各种方式（从 C++ 和 JavaScript）。

接下来（在 PropagatePendingExceptionToExternalTryCatch 中）我们有：
```c++
  // 获取最顶层的 JS_ENTRY handler，如果不存在则不能在顶部。
  Address entry_handler = Isolate::handler(thread_local_top());                 
  if (entry_handler == kNullAddress) return false;
```
接下来，我们有：
```c++
  if (!IsExternalHandlerOnTop(exception)) {                                     
    thread_local_top()->external_caught_exception_ = false;                     
    return true;                                                                
  }
```
我目前对这些不同的 handler 感到非常困惑，我们有一个用于……
