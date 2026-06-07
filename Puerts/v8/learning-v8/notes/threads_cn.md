### V8 中的线程
当 V8 执行 JavaScript 时，它使用单线程，因此所有 JavaScript 函数将被推入同一个 call stack（在单个 isolate 内）。

但我们可以看到，当 V8 启动时，它会分配线程。

如果我们查看 hello_world 示例，我们可以看到我们创建了一个新的 Platform：
```cpp
  std::unique_ptr<Platform> platform = platform::NewDefaultPlatform();
```
`NewDefaultPlatform` 在 `include/libplatform/libplatform.h` 中声明，并创建一个新的 [DefaultPlatform](#defaultplatform)。

### Platform
`Platform` 在 `include/v8-platform.h` 中声明。这个抽象类的预期用户是 embedder，他们需要扩展它。但在 `src/libplatform/default-platform.h` 中也有一个默认实现。

```cpp
class Platform {                                                                
 public:
  ...
  virtual int NumberOfWorkerThreads() = 0;
  virtual std::shared_ptr<v8::TaskRunner> GetForegroundTaskRunner(Isolate* isolate) = 0;
  virtual void CallOnWorkerThread(std::unique_ptr<Task> task) = 0; 
  virtual std::unique_ptr<JobHandle> PostJob(TaskPriority priority,
      std::unique_ptr<JobTask> job_task) {               
    return nullptr;
};
```

### DefaultPlatform
这是 V8 自带的默认 Platform 实现。

```cpp
class V8_PLATFORM_EXPORT DefaultPlatform : public NON_EXPORTED_BASE(Platform) { 
 public:                                                                           
  explicit DefaultPlatform(                                                        
      int thread_pool_size = 0,                                                    
      IdleTaskSupport idle_task_support = IdleTaskSupport::kDisabled,              
      std::unique_ptr<v8::TracingController> tracing_controller = {}); 
 private:
  std::shared_ptr<DefaultWorkerThreadsTaskRunner> worker_threads_task_runner_;

  std::map<v8::Isolate*, std::shared_ptr<DefaultForegroundTaskRunner>>
      foreground_task_runner_map_;
```

`IdleTaskSupport` 是 `libplatform.h` 中定义的一个枚举：
```cpp
enum class IdleTaskSupport { kDisabled, kEnabled };
```
我们可以看到，如果未指定，默认为 `kDisabled`。
我目前最感兴趣的是如何使用线程以及线程的用途，最大线程数是：
```cpp
constexpr int kMaxThreadPoolSize = 16;
```
如果未显式指定此值，如上所示，默认值为 0，在这种情况下，线程数将为处理器数 -1（或 kMaxThreadPoolSize）。

如果我们查看 DefaultPlatform 的私有成员，我们可以看到它有一个名为 `worker_threads_task_runner_` 的字段，类型为 `DefaultWorkerThreadsTaskRunner`，该类在 `src/libplatform/default-worker-threads-task-runner.h` 中声明：
```cpp
class V8_PLATFORM_EXPORT DefaultWorkerThreadsTaskRunner                            
    : public NON_EXPORTED_BASE(TaskRunner) {
 public:
  using TimeFunction = double (*)();
  DefaultWorkerThreadsTaskRunner(uint32_t thread_pool_size, TimeFunction time_function);
  ...
 private:
  class WorkerThread : public base::Thread {
   public:
    explicit WorkerThread(DefaultWorkerThreadsTaskRunner* runner);
    ~WorkerThread() override;
    // 此线程尝试在循环中从 |runner_| 获取 tasks 并运行它们。
    void Run() override;
   private:
    DefaultWorkerThreadsTaskRunner* runner_;
    DISALLOW_COPY_AND_ASSIGN(WorkerThread);
  };

  std::vector<std::unique_ptr<WorkerThread>> thread_pool_; 
}
```
我们可以在 `src/libplatform/default-worker-threads-task-runner.cc` 中找到具体实现：
```cpp
DefaultWorkerThreadsTaskRunner::DefaultWorkerThreadsTaskRunner(
    uint32_t thread_pool_size, TimeFunction time_function)
    : queue_(time_function), time_function_(time_function) {
  for (uint32_t i = 0; i < thread_pool_size; ++i) {
    thread_pool_.push_back(std::make_unique<WorkerThread>(this));
  }
}
```
注意，上述代码将创建 `thread_pool_size` 个 `WorkerThread` 实例并将它们放入向量 `thread_pool_` 中。
我们可以看到 WorkerThread 继承自 `base::Thread`，该基类在 `src/base/platform/platform.h` 中声明。

现在，什么类型的 tasks 会被提交到这个线程池？
一个例子是编译 wasm 时：
```cpp
  V8::GetCurrentPlatform()->CallOnWorkerThread(std::move(task));
```
此外，还有来自 heap 命名空间的许多调用，涉及 garbage collection。

### TaskRunner
在 `include/v8-platform.h` 中声明：
```cpp
class Task {
 public:
  virtual ~Task() = default;
  virtual void Run() = 0;
};

class TaskRunner {
 public:
  virtual void PostTask(std::unique_ptr<Task> task) = 0; 
  ...
```

### TracingController
TODO: 这可能在单独的笔记文件中更合适。

```console
 ../v8_src/v8/out/x64.release_gcc/d8 --enable-tracing -e 'console.log("bajja");'
```
现在我们可以打开 `chrome://inspect` 并加载使用 `--enable-tracing` 选项生成的名为 `v8_trace.json` 的文件。
V8 tracing 文档的 [链接](https://v8.dev/docs/trace)。
