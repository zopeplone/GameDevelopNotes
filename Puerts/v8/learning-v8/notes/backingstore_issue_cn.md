### Backingstore 问题
本文档包含在 Node.js 中发现的一个问题的笔记。

Node.js issue: https://github.com/nodejs/node/issues/35669

运行此 [测试](https://github.com/nodejs/node/blob/11f1ad939f902ddbeb406350a6162359e04c13ec/test/cctest/test_environment.cc#L349) 时生成以下 address sanitizer 错误：
```console
$ out/Release/cctest --gtest_filter=EnvironmentTest.BufferWithFreeCallbackIsDetached
Running main() from ../test/cctest/gtest/gtest_main.cc
Note: Google Test filter = EnvironmentTest.BufferWithFreeCallbackIsDetached
[==========] Running 1 test from 1 test suite.
[----------] Global test environment set-up.
[----------] 1 test from EnvironmentTest
[ RUN      ] EnvironmentTest.BufferWithFreeCallbackIsDetached
6
=================================================================
==2773765==ERROR: AddressSanitizer: new-delete-type-mismatch on 0x60400001af50 in thread T0:
  object passed to delete has wrong type:
  size of the allocated type:   48 bytes;
  size of the deallocated type: 1 bytes.
    #0 0x7f0a3a306175 in operator delete(void*, unsigned long) (/lib64/libasan.so.5+0x111175)
    #1 0x38328c2 in v8::internal::JSArrayBuffer::Detach(bool) ...
    #2 0x29bf88a in v8::ArrayBuffer::Detach() ...
    #3 0x151adae in node::Buffer::(anonymous namespace)::CallbackInfo::CleanupHook(void*) ...
    ...
```

注意调试器中的以下行：
```console
* thread #1, name = 'cctest', stop reason = Deallocation size different from allocation size
```
因此 asan 指出被删除的类型大小不匹配，这是正确的。我将在下面详细介绍，但简而言之，asan 会记录 `v8::internal::BackingStore` 实例（48 字节类型）的创建时间，稍后当对该内存分配的指针调用 delete 时，大小将是 1（`v8::BackingStore`）。

我们可以通过设置以下内容来绕过测试中的此错误：
```cpp
extern "C" const char* __asan_default_options() {
  return "new_delete_type_mismatch=0";
}
```

以下是调查此问题的笔记。

虽然结合 Node 测试 fixture 来理解有些困难，但在 [backingstore_test.cc](../test/backingstore_test.cc) 中有一个独立的示例。
我们将重点关注该示例的以下部分：
```cpp
  Local<ArrayBuffer> ab;
  {
    std::unique_ptr<BackingStore> bs = ArrayBuffer::NewBackingStore(
        data, sizeof(data), BackingStoreDeleter, &deleter_called);
    ab = ArrayBuffer::New(isolate_, std::move(bs));
    std::cout << "ArrayBuffer created\n";
  }
  ab->Detach();
```
如果我们查看 `src/api/api.cc` 中的 `v8::ArrayBuffer::NewBackingStore`，有以下内容：
```cpp
std::unique_ptr<v8::BackingStore> v8::ArrayBuffer::NewBackingStore(
    void* data, size_t byte_length, v8::BackingStore::DeleterCallback deleter,
    void* deleter_data) {
  CHECK_LE(byte_length, i::JSArrayBuffer::kMaxByteLength);
  std::unique_ptr<i::BackingStoreBase> backing_store =
      i::BackingStore::WrapAllocation(data, byte_length, deleter, deleter_data,
                                      i::SharedFlag::kNotShared);
  return std::unique_ptr<v8::BackingStore>(
      static_cast<v8::BackingStore*>(backing_store.release()));
}
```
注意，`unique_ptr` 的类型是 `v8::internal::BackingStoreBase`，而 `WrapAllocation` 返回 `std::unique_ptr<v8::internal::BackingStore>`：
```cpp
std::unique_ptr<BackingStore> BackingStore::WrapAllocation(
    void* allocation_base, size_t allocation_length,
    v8::BackingStore::DeleterCallback deleter, void* deleter_data,
    SharedFlag shared) {
  bool is_empty_deleter = (deleter == v8::BackingStore::EmptyDeleter);
  auto result = new BackingStore(allocation_base,    // start
                                 allocation_length,  // length
                                 allocation_length,  // capacity
                                 shared,             // shared
                                 false,              // is_wasm_memory
                                 true,               // free_on_destruct
                                 false,              // has_guard_regions
                                 true,               // custom_deleter
                                 is_empty_deleter);  // empty_deleter
  result->type_specific_data_.deleter = {deleter, deleter_data};
  TRACE_BS("BS:wrap   bs=%p mem=%p (length=%zu)\n", result,
           result->buffer_start(), result->byte_length());
  return std::unique_ptr<BackingStore>(result);
}
```
`WrapAllocation` 创建一个新的 `v8::internal::BackingStore`，然后在 unique_ptr 中返回，但请记住，存储该指针的类型是 `std::unique_ptr<i::BackingStoreBase>`。
然后由 `ArrayBuffer::NewBackingStore` 将其转换为 `v8::BackingStore` 后返回：
```cpp
  std::unique_ptr<i::BackingStoreBase> backing_store =
      i::BackingStore::WrapAllocation(data, byte_length, deleter, deleter_data,
                                      i::SharedFlag::kNotShared);
  return std::unique_ptr<v8::BackingStore>(
      static_cast<v8::BackingStore*>(backing_store.release()));
```
注意，`backing_store` 的类型是 __v8::internal::BackingStoreBase__，因此我们在这里使用基类。然后将其转换为 `v8::BackingStore`。

接下来，返回的 `v8::BackingStore` 将被传入 `ArrayBuffer::New`：
```cpp
  Local<ArrayBuffer> ab;
  {
    std::unique_ptr<BackingStore> bs = ArrayBuffer::NewBackingStore(
        data, data_len, BackingStoreDeleter, nullptr);
    ab = ArrayBuffer::New(isolate_, std::move(bs));
```
然后 backingstore 被移动到 `ArrayBuffer::New`（在 `src/api/api.cc` 中）：
```cpp
  std::shared_ptr<i::BackingStore> i_backing_store(ToInternal(std::move(backing_store)));
  i::Handle<i::JSArrayBuffer> obj = i_isolate->factory()->NewJSArrayBuffer(std::move(i_backing_store));
```
`ToInternal` 如下所示：
```cpp
std::shared_ptr<i::BackingStore> ToInternal(
    std::shared_ptr<i::BackingStoreBase> backing_store) {
  return std::static_pointer_cast<i::BackingStore>(backing_store);
}
```
注意，参数类型是 `std::shared_ptr<i::BackingStoreBase>`，而传入的类型将是 `std::unique_ptr<v8::BackingStore>`。然后将其转换为 `std::shared_ptr<i::BackingStore>`。

`NewJSArrayBuffer` 将调用 `js-array-buffer.cc` 中的 `Attach`：
```cpp
void JSArrayBuffer::Attach(std::shared_ptr<BackingStore> backing_store) {
  Isolate* isolate = GetIsolate();
  set_backing_store(isolate, backing_store->buffer_start());
  set_byte_length(backing_store->byte_length());
  if (backing_store->is_wasm_memory()) set_is_detachable(false);
  if (!backing_store->free_on_destruct()) set_is_external(true);
  Heap* heap = isolate->heap();
  ArrayBufferExtension* extension = EnsureExtension();
  size_t bytes = backing_store->PerIsolateAccountingLength();
  extension->set_accounting_length(bytes);
  extension->set_backing_store(std::move(backing_store));
  heap->AppendArrayBufferExtension(*this, extension);
}
```
现在，稍后某个时刻 `ab->Detach()` 将被调用，这将进入 `src/objects/js-array-buffer.cc`：
```cpp
void JSArrayBuffer::Detach(bool force_for_wasm_memory) {
  ...
  Isolate* const isolate = GetIsolate();
  if (backing_store()) {
    std::shared_ptr<BackingStore> backing_store;
    backing_store = RemoveExtension();
    CHECK_IMPLIES(force_for_wasm_memory, backing_store->is_wasm_memory());
  }
  ...
}
```
