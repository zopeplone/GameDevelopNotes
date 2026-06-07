### Tasks 和 Micro tasks
本文档更仔细地研究 tasks 和 microtasks。

### Tasks
正如 [介绍](./intro_cn.md) 中提到的，task 是被放入 task/callback queue 中的东西，将由运行时取出并放入 execution stack。
例如，如果一个函数调用 `setTimeout`，一旦计时器到期，将把要执行的函数放入 task/callback queue。

让我们看一个使用 Node.js 的具体 [示例](../lib/task.js)：
```console
$ env NODE_DEBUG=timer node lib/task.js 
main...
TIMER 3924762: no 1 list was found in insert, creating a new one
main...done
TIMER 3924762: process timer lists 145
TIMER 3924762: timeout callback 1
something
TIMER 3924762: 1 list empty
```
在 Node.js 中，`setTimeout` 是使用 `libuv` 的 `uv_timer_start` 实现的。

### Microtasks
[介绍](./intro_cn.md) 中有一些关于 microtasks 的背景信息。本页将深入研究其内部机制。

在 [microtask_test](../test/microtask_test.cc) 中有一个运行 microtask 的示例。

Isolate 类暴露了许多与入队和运行 microtasks 相关的函数：
```cpp
class V8_EXPORT Isolate {                                                       
 public:     
 ...
  V8_DEPRECATE_SOON("Use PerformMicrotaskCheckpoint.")                          
  void RunMicrotasks() { PerformMicrotaskCheckpoint(); }  
  void PerformMicrotaskCheckpoint();
  void EnqueueMicrotask(Local<Function> microtask);
  void EnqueueMicrotask(MicrotaskCallback callback, void* data = nullptr);
};
```

让我们看看当我们将一个 microtask 函数入队时会发生什么：
```console
$ lldb -- ./test/microtask_test
(lldb) br s -f microtask_test.cc -l 33
(lldb) r
```
`Isolate::EnqueueMicrotask` 可以在 `src/api/api.cc` 中找到。
实际的调用最终会进入 `microtask-queue.cc`，位于 `src/execution/microtask-queue.cc`：
```cpp
void MicrotaskQueue::EnqueueMicrotask(v8::Isolate* v8_isolate,
                                      v8::Local<Function> function) {
  Isolate* isolate = reinterpret_cast<Isolate*>(v8_isolate);
  HandleScope scope(isolate);
  Handle<CallableTask> microtask = isolate->factory()->NewCallableTask(
      Utils::OpenHandle(*function), isolate->native_context());
  EnqueueMicrotask(*microtask);
}
```
`NewCallableTask` 可以在 `src/heap/factory.cc` 中找到。
