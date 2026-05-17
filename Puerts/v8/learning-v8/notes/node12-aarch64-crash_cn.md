### Node.js Yarn 在 ARM 64 上的问题
本节记录了 Node.js v12.16.1 与 yarn 的问题排查过程。

Backtrace：
```console
(gdb) bt 10
#0  0x0000aaaaab4d07f4 in std::_Rb_tree<unsigned long, unsigned long, std::_Identity<unsigned long>, std::less<unsigned long>, std::allocator<unsigned long> >::_M_get_insert_unique_pos (__k=<synthetic pointer>: <optimized out>, this=0x18)
    at /usr/include/c++/8/bits/stl_tree.h:2101
#1  std::_Rb_tree<unsigned long, unsigned long, std::_Identity<unsigned long>, std::less<unsigned long>, std::allocator<unsigned long> >::_M_insert_unique<unsigned long const&> (__v=<synthetic pointer>: <optimized out>, this=0x18)
    at /usr/include/c++/8/bits/stl_tree.h:2109
#2  std::set<unsigned long, std::less<unsigned long>, std::allocator<unsigned long> >::insert (
    __x=<synthetic pointer>: <optimized out>, this=0x18) at /usr/include/c++/8/bits/stl_set.h:511
#3  v8::internal::CodeObjectRegistry::RegisterNewlyAllocatedCodeObject (this=0x0, code=code@entry=1301151744)
    at ../deps/v8/src/heap/spaces.cc:620
#4  0x0000aaaaab47ea90 in v8::internal::Heap::AllocateRaw (alignment=v8::internal::kWordAligned, 
    origin=v8::internal::AllocationOrigin::kRuntime, type=-80, size_in_bytes=129152, this=0xaaaaacbbaab0)
    at ../deps/v8/src/objects/heap-object.h:108
#5  v8::internal::Heap::AllocateRawWithLightRetry (this=this@entry=0xaaaaacbbaab0, size=size@entry=129152, 
    allocation=allocation@entry=v8::internal::AllocationType::kCode, 
    origin=origin@entry=v8::internal::AllocationOrigin::kRuntime, alignment=alignment@entry=v8::internal::kWordAligned)
    at ../deps/v8/src/heap/heap.cc:4886
#6  0x0000aaaaab47ec38 in v8::internal::Heap::AllocateRawWithRetryOrFail (this=0xaaaaacbbaab0, size=129152, 
    allocation=v8::internal::AllocationType::kCode, origin=v8::internal::AllocationOrigin::kRuntime, 
    alignment=v8::internal::kWordAligned) at ../deps/v8/src/heap/heap.cc:4908
#7  0x0000aaaaab451fe0 in v8::internal::Heap::AllocateRawWithRetryOrFail (alignment=v8::internal::kWordAligned, 
    allocation=v8::internal::AllocationType::kCode, size=129152, this=0xaaaaacbbaab0) at ../deps/v8/src/heap/heap.h:1762
#8  v8::internal::Factory::CodeBuilder::BuildInternal (this=this@entry=0xffffffff96b0, 
    retry_allocation_or_fail=retry_allocation_or_fail@entry=true) at ../deps/v8/src/heap/factory.cc:121
#9  0x0000aaaaab452380 in v8::internal::Factory::CodeBuilder::Build (this=this@entry=0xffffffff96b0)
    at ../deps/v8/src/heap/factory.cc:205
```
让我们看看 `AllocateRawWithLightRetry`：
```c++
4882 HeapObject Heap::AllocateRawWithLightRetry(int size, AllocationType allocation, 
4883                                            AllocationOrigin origin,             
4884                                            AllocationAlignment alignment) {     
4885   HeapObject result;                                                            
4886   AllocationResult alloc = AllocateRaw(size, allocation, origin, alignment);
```
再往下一帧是：
```
(gdb) down
#4  0x0000aaaaab47ea90 in v8::internal::Heap::AllocateRaw (alignment=v8::internal::kWordAligned, 
    origin=v8::internal::AllocationOrigin::kRuntime, type=-80, size_in_bytes=129152, this=0xaaaaacbbaab0)
    at ../deps/v8/src/objects/heap-object.h:108
108   inline Address address() const { return ptr() - kHeapObjectTag; }
```
让我们在 heap.cc:4886 处设置条件断点：
```console
(gdb) br heap.cc:4886 if size == 129152
Breakpoint 5 at 0xaaaaab47e700: file ../deps/v8/src/heap/heap.cc, line 4886.
```
这将带我们进入 heap-inl.h：
```c++
206   } else if (AllocationType::kCode == type) {                                   
207     if (size_in_bytes <= code_space()->AreaSize() && !large_object) {           
208       allocation = code_space_->AllocateRawUnaligned(size_in_bytes);             
209     } else {                                                                    
210       allocation = code_lo_space_->AllocateRaw(size_in_bytes);                   
211     }                                                         
```
在我们的情况下，将执行第 210 行。这将带我们进入 `spaces.cc`：
```c++
4386 AllocationResult CodeLargeObjectSpace::AllocateRaw(int object_size) {           
4387   return LargeObjectSpace::AllocateRaw(object_size, EXECUTABLE);                 
4388 } 
```
在 `heap-inl.h` 和 `AllocateRaw` 中，函数末尾附近的代码：
```c++
227   if (allocation.To(&object)) {                                                 
228     if (AllocationType::kCode == type) {                                         
229       // 如果内存块的 MemoryChunk 尚未取消保护，则取消保护。
230       UnprotectAndRegisterMemoryChunk(object);                                   
231       ZapCodeObject(object.address(), size_in_bytes);                            
232       if (!large_object) {                                                       
233         MemoryChunk::FromHeapObject(object)                                      
234             ->GetCodeObjectRegistry()                                            
235             ->RegisterNewlyAllocatedCodeObject(object.address());                
236       }                                                                          
237     }                                                                            
238     OnAllocationEvent(object, size_in_bytes);                                    
239   }                                                                              
240                                                                                  
241   return allocation;  
```
这将调用 `MemoryChunk::FromHeapObject(object)->GetCodeObjectRegistry()->RegisterNewlyAllocatedCodeObject(object.address())`。

MemoryChunk::FromHeapObject 只是一个 reinterpret_cast：
```c++
return reinterpret_cast<MemoryChunk*>(object.ptr() & ~kPageAlignmentMask);

const int kPageSizeBits = 18;
static const intptr_t kPageAlignmentMask = (intptr_t{1} << kPageSizeBits) - 1;
```
注意 backtrace 中的 `this=0x18`：
```console
0x0000aaaaab4d07f4 in std::_Rb_tree<unsigned long, unsigned long, std::_Identity<unsigned long>, std::less<unsigned long>, std::allocator<unsigned long> >::_M_get_insert_unique_pos (__k=<synthetic pointer>: 716832768, this=0x18)
```
我第一次看到这个以及段错误来自 `std::set::insert` 时感到很奇怪，但希望以下示例能说明问题：
```c++
#include <stdint.h>
#include <set>
#include <iostream>

struct something {
  std::set<int> xs;
};

int main(int argc, char** argv) {
  intptr_t i = intptr_t{1} + 17;
  struct something* s = reinterpret_cast<something*>(i);
  std::cout << i << '\n';
  s->xs.insert(2);
  return 0;
}
```
这就解释了 `this` 的奇怪值，以及为什么即使 this 无效也能调用 insert：
```
auto result = code_object_registry_newly_allocated_.insert(code);
```
因此我们现在理解了段错误的原因。但我们仍需要理解为什么会发生这种情况。

让我们创建断点以便可以轻松复现此问题：
```console
(gdb) br heap-inl.h:210 if size_in_bytes == 129152
(gdb) br heap-inl.h:233 if size_in_bytes == 129152
```
第一个断点是 `allocation` 通过调用 `code_lo_space` 上的 `AllocateRaw` 创建的位置（其类型为 `CodeLargeObjectSpace`，可在 `src/heap/spaces.h` 中找到）：
```c++
210       allocation = code_lo_space_->AllocateRaw(size_in_bytes);                  
```
`allocation` 之前声明为：
```c++
  AllocationResult allocation;
```
`AllocationResult` 实例有一个单一成员，类型为 Object。
有一个函数可以将对象转换为某种类型的指针：
```c++
template <typename T>                                                             
bool To(T* obj) {                                                                 
  if (IsRetry()) return false;                                                
  *obj = T::cast(object_);                                                    
  return true;                                                                
}
```
此函数在以下 `if` 语句中被调用：
```c++
  HeapObject object; 
  ...
  if (allocation.To(&object)) {
    if (AllocationType::kCode == type) {
      ...
}
```
`object_` 字段的值是：
```console
(gdb) p allocation.object_
$10 = {<v8::internal::TaggedImpl<(v8::internal::HeapObjectReferenceType)1, unsigned long>> = {static kIsFull = true, 
    static kCanBeWeak = false, ptr_ = 1332346881}, static kHeaderSize = 0}
```
不幸的是，object 的值被优化掉了：
```console
(gdb) p object
$13 = <optimized out>
```
所以这是将传递给 `HeapObject::cast(object_)` 的对象。此 `To` 函数将返回 true，然后下一行将执行：
```c++
      UnprotectAndRegisterMemoryChunk(object);
      ZapCodeObject(object.address(), size_in_bytes);
      if (!large_object) {
        MemoryChunk::FromHeapObject(object)
            ->GetCodeObjectRegistry()
            ->RegisterNewlyAllocatedCodeObject(object.address());
      }
```
我尝试在 [heap_test](https://github.com/danbev/learning-v8/blob/f3aae7a4e073f5a3183199aab89b0a8abe09680f/test/heap_test.cc#L48) 中复现此问题。

看起来 `code_object_registry_` 在这种情况下没有被创建。在 `MemoryChunk::Initialize` 中有以下检查（在 `deps/v8/src/heap/spaces.cc` 中），该函数由我们之前展示的 `CodeLargeObjectSpace::AllocateRaw` 调用：
```c++
MemoryChunk* MemoryChunk::Initialize(BasicMemoryChunk* basic_chunk, Heap* heap, 
                                     Executability executable) {   
  ...

  if (chunk->owner()->identity() == CODE_SPACE) {
    chunk->code_object_registry_ = new CodeObjectRegistry();
  } else {
    chunk->code_object_registry_ = nullptr;
```
