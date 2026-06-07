### Heap
本文档将尝试描述 V8 中的 Heap 以及相关的类和函数。

当创建 V8 Isolate 时，通常通过类似以下方式完成：
```cpp
  isolate_ = v8::Isolate::New(create_params_);
```
`v8::Isolate::New` 中发生的第一件事是分配 Isolate：
```cpp
Isolate* Isolate::New(const Isolate::CreateParams& params) {
  Isolate* isolate = Allocate()
  ...
}
```
而 Allocate 调用 `v8::internal::Isolate::New` 并将其转换回 `v8::Isolate`：
```cpp
Isolate* Isolate::Allocate() {
  return reinterpret_cast<Isolate*>(i::Isolate::New());
}
```

### IsolateAllocator
每个 Isolate 都有一个关联的 IsolateAllocator 实例，该实例被传入构造函数：
```cpp
Isolate* Isolate::New(IsolateAllocationMode mode) {
  std::unique_ptr<IsolateAllocator> isolate_allocator =
      std::make_unique<IsolateAllocator>(mode);
  void* isolate_ptr = isolate_allocator->isolate_memory();
  Isolate* isolate = new (isolate_ptr) Isolate(std::move(isolate_allocator));
  ...
  return isolate;
}
```
注意，上述代码使用了 placement new，并指定了新 Isolate 应构造的内存地址（isolate_ptr）。

IsolateAllocator 的定义可以在 `src/init/isolate-allocator.h` 中找到，以下是我认为有趣的几个方法和字段：
```cpp
class V8_EXPORT_PRIVATE IsolateAllocator final {
 public:
  void* isolate_memory() const { return isolate_memory_; }
  v8::PageAllocator* page_allocator() const { return page_allocator_; }

 private:
  Address InitReservation();
  void CommitPagesForIsolate(Address heap_reservation_address);

  // 为 Isolate 实例分配的内存。
  void* isolate_memory_ = nullptr;
  v8::PageAllocator* page_allocator_ = nullptr;
  std::unique_ptr<base::BoundedPageAllocator> page_allocator_instance_;
  VirtualMemory reservation_;
};
```

Isolate 有一个 `page_allocator()` 方法，返回 IsolateAllocator 中的 `v8::PageAllocator`：
```cpp
v8::PageAllocator* Isolate::page_allocator() {
  return isolate_allocator_->page_allocator();
}
```
我们正处于 `Isolate::New(IsolationAllocationMode)` 中，下一行是：
```cpp
  std::unique_ptr<IsolateAllocator> isolate_allocator =
      std::make_unique<IsolateAllocator>(mode);
```

默认的 allocation mode 是 `IsolateAllocationMode::kInV8Heap`：
```cpp
  static Isolate* New(
      IsolateAllocationMode mode = IsolateAllocationMode::kDefault);
```
mode 的可选值为：
```cpp
  // 在 C++ heap 中使用默认的 new/delete 运算符分配 Isolate。
  kInCppHeap,

  // 在 V8 heap reservation 内的已提交区域中分配 Isolate。
  kInV8Heap,

#ifdef V8_COMPRESS_POINTERS
  kDefault = kInV8Heap,
#else
  kDefault = kInCppHeap,
#endif
};
```

构造函数接受如上所述的 AllocationMode：
```cpp
IsolateAllocator::IsolateAllocator(IsolateAllocationMode mode) {
#if V8_TARGET_ARCH_64_BIT
  if (mode == IsolateAllocationMode::kInV8Heap) {
    Address heap_reservation_address = InitReservation();
    CommitPagesForIsolate(heap_reservation_address);
    return;
  }
#endif  // V8_TARGET_ARCH_64_BIT
```
让我们仔细看看 `InitReservation`（`src/init/isolate-allocator.cc`）：
```cpp
Address IsolateAllocator::InitReservation() {
  v8::PageAllocator* platform_page_allocator = GetPlatformPageAllocator();
  ...
   VirtualMemory padded_reservation(platform_page_allocator,
                                    reservation_size * 2,
                                    reinterpret_cast<void*>(hint));

  ...
  reservation_ = std::move(reservation);
```
`GetPlatformPageAllocator()` 可以在 `src/utils/allocation.cc` 中找到：
```cpp
v8::PageAllocator* GetPlatformPageAllocator() {
  DCHECK_NOT_NULL(GetPageTableInitializer()->page_allocator());
  return GetPageTableInitializer()->page_allocator();
}
```
现在，`GetPageTableInitializer` 也在 `allocation.cc` 中定义，但使用了宏：
```cpp
DEFINE_LAZY_LEAKY_OBJECT_GETTER(PageAllocatorInitializer,
                                GetPageTableInitializer)
```
此宏可以在 `src/base/lazy-instance.h` 中找到：
```cpp
#define DEFINE_LAZY_LEAKY_OBJECT_GETTER(T, FunctionName, ...) \
  T* FunctionName() {                                         \
    static ::v8::base::LeakyObject<T> object{__VA_ARGS__};    \
    return object.get();                                      \
  }
```
因此预处理器会将上述用法展开为：
```cpp
  PageAllocatorInitializer* GetPageTableInitializer() {
    static ::v8::base::LeakyObject<PageAllocatorInitializer> object{};
    return object.get();
  }
```
```cpp
PageAllocatorInitializer() {
    page_allocator_ = V8::GetCurrentPlatform()->GetPageAllocator();
    if (page_allocator_ == nullptr) {
      static base::LeakyObject<base::PageAllocator> default_page_allocator;
      page_allocator_ = default_page_allocator.get();
    }
  }
```
请注意，这里调用了 `V8::GetCurrentPlatform()`，回想一下，platform 在 Isolate 创建之前就已创建和初始化：
```cpp
    platform_ = v8::platform::NewDefaultPlatform();
    v8::V8::InitializePlatform(platform_.get());
    v8::V8::Initialize();
```
再次查看 PageAllocator 的构造函数，我们看到它从 platform 中获取 PageAllocator。这个 PageAllocator 是如何/何时创建的？

它在 `DefaultPlatform` 的构造函数（`src/libplatform/default-platform.cc`）中创建，由 `NewDefaultPlatform` 调用：
```cpp
DefaultPlatform::DefaultPlatform(
    int thread_pool_size, IdleTaskSupport idle_task_support,
    std::unique_ptr<v8::TracingController> tracing_controller)
    : thread_pool_size_(GetActualThreadPoolSize(thread_pool_size)),
      idle_task_support_(idle_task_support),
      tracing_controller_(std::move(tracing_controller)),
      page_allocator_(std::make_unique<v8::base::PageAllocator>()) {
  ...
}
```
PageAllocator 的构造函数可以在 `src/base/page-allocator.cc` 中找到：
```cpp
PageAllocator::PageAllocator()
    : allocate_page_size_(base::OS::AllocatePageSize()),
      commit_page_size_(base::OS::CommitPageSize()) {}
```
`allocate_page_size_` 是通过调用（在 Linux 上）获取的：
```cpp
size_t OS::AllocatePageSize() {
  return static_cast<size_t>(sysconf(_SC_PAGESIZE));
}
```
这是物理内存中 page frame 单位的固定字节块数。（_SC 是 sysconf 函数的 System Config 参数前缀）。
page 是存储在 page frame 中的单位：
```
                             Physical Memory                                          
                         +---------------------------+                                  
      physical address 1 | PageFrame 1: page content |                                  
                         +---------------------------+                                  
      physical address 2 | PageFrame 2: page content |                                  
                         +---------------------------+                                  
```
