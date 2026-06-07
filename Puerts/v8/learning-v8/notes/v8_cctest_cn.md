### V8 单元测试
V8 的单元测试不像本项目那样使用 gtest，本文档包含关于 `CcTest` 如何工作的笔记。

### CcTest
这是测试驱动程序，即它注册和运行测试，可以在 `test/cctest/cctest.h` 中找到：
```cpp
class CcTest {
 public:
  using TestFunction = void();
  CcTest(TestFunction* callback, const char* file, const char* name,
         bool enabled, bool initialize);
```
因此，要注册一个测试，可以使用上述构造函数，但定义了宏来简化此过程。例如，要注册一个名为 `DoSomething` 的测试，只需编写：
```cpp
TEST(DoSomething) {
  ...
}
```
它将被预处理器展开为：
```cpp
CcTest register_test_DoSomething(TestDoSomething, __FILE__, DoSomething, true, true) {
  ...
}
```
还有像 `DISABLED_TEST`、`UNINITIALIZED_TEST` 这样的宏。

CcTest 还包含辅助函数，用于访问内部的 Isolate、Heap、调用 garbage collection 等。

### 运行单个测试
要列出所有测试，可以使用以下命令：
```console
$ out/learning_v8/cctest --list
```
从中你可以用 grep 搜索名称，然后使用该名称：
```console
$ out/learning_v8/cctest test-heap/CodeLargeObjectSpace
```

### Gtest
V8 也有使用 Google Test 而非 CcTest "驱动程序" 的测试。

运行单个测试：
```console
$ env ASAN_OPTIONS="detect_odr_violation=0" ./out/learning_v8/unittests --gtest_filter=BackingStoreTest.CopyWasmMemory
```

有时可能需要使某些函数对测试可用。例如，我在向 `test/cctest/heap/test-heap.cc` 添加测试时遇到了这种情况。有一个头文件 `test/cctest/heap/heap-tester.h`，其中包含以下宏：
```cpp
// 应有权访问 {v8::internal::Heap} 私有方法的测试。
// 这些测试需要使用 HEAP_TEST(Name) { ... } 来定义。
#define HEAP_TEST_METHODS(V)                                \
  V(CodeLargeObjectSpace)                                   \
  V(CompactionFullAbortedPage)                              \
  ...
```
这将可以编译并手动运行测试，但除非所使用的函数也使用 `V8_EXPORT_PRIVATE` 声明，否则可能在 CI 上失败。

使用 stress 选项运行测试：
```console
$ out/learning_v8/cctest test-heap/CodeLargeObjectSpace --random-seed=768076620 --stress-incremental-marking --nohard-abort --testing-d8-test-runner
```

### Address sanitizer
要启用 address sanitizer，首先安装 asan：
```console
$ sudo dnf install libasan libasan-static
```
然后在 build args 中添加 `is_asan=true`，在我的情况下是 `out/learning_v8/args.gn`：
```console
is_asan = true
is_clang = true
```
注意，还必须设置 `is_clang`，否则会出错：
```console
[1/1] Regenerating ninja files
FAILED: build.ninja
../../buildtools/linux64/gn --root=../.. -q gen .
ERROR at //build/config/sanitizers/sanitizers.gni:195:1: Assertion failed.
assert(!using_sanitizer || is_clang,
^-----
Sanitizers (is_*san) require setting is_clang = true in 'gn args'
See //build/config/v8_target_cpu.gni:5:1: whence it was imported.
```

#### 参数化测试
本节展示了一个 V8 中参数化测试的示例，并记录了我使用 gcc 看到的一个编译器警告。

涉及的测试是 `TEST_P(InstructionSelectorChangeInt32ToInt64Test, ChangeInt32ToInt64WithLoad)`，位于 `test/unittests/compiler/x64/instruction-selector-x64-unittest.cc`：
```cpp
namespace {
struct LoadWithToInt64Extension {
  MachineType type;
  ArchOpcode expected_opcode;
};

static const LoadWithToInt64Extension kLoadWithToInt64Extensions[] = {
    {MachineType::Int8(), kX64Movsxbq},
    {MachineType::Uint8(), kX64Movzxbq},
    {MachineType::Int16(), kX64Movsxwq},
    {MachineType::Uint16(), kX64Movzxwq},
    {MachineType::Int32(), kX64Movsxlq}};

}  // namespace

using InstructionSelectorChangeInt32ToInt64Test =
    InstructionSelectorTestWithParam<LoadWithToInt64Extension>;

TEST_P(InstructionSelectorChangeInt32ToInt64Test, ChangeInt32ToInt64WithLoad) {
  const LoadWithToInt64Extension extension = GetParam();
  StreamBuilder m(this, MachineType::Int64(), MachineType::Pointer());
  m.Return(m.ChangeInt32ToInt64(m.Load(extension.type, m.Parameter(0))));
  Stream s = m.Build();
  ASSERT_EQ(1U, s.size());
  EXPECT_EQ(extension.expected_opcode, s[0]->arch_opcode());
}

INSTANTIATE_TEST_SUITE_P(InstructionSelectorTest,
                         InstructionSelectorChangeInt32ToInt64Test,
                         ::testing::ValuesIn(kLoadWithToInt64Extensions));
```
注意 `using` 语句，我们可以在 `test/unittests/compiler/backend/instruction-selector-unittest.h` 中找到 `InstructionSelectorTestWithParam`：
```cpp
template <typename T>
class InstructionSelectorTestWithParam
    : public InstructionSelectorTest,
      public ::testing::WithParamInterface<T> {};
```
还要注意 `T` 是 `LoadWithToInt64Extension`，并且它在匿名命名空间中（稍后会详细说明）。
`INSTANTIATE_TEST_SUITE_P` 将从 `InstructionSelectorTest` 实例化测试。

在 [parameterized_test.cc](test/parameterized_test.cc) 中有一个示例。

现在，回到匿名命名空间。当前正在生成以下编译器警告：
```console
In file included from ../../third_party/googletest/src/googletest/include/gtest/gtest-param-test.h:181,
                 from ../../third_party/googletest/src/googletest/include/gtest/gtest.h:67,
                 from ../../testing/gtest/include/gtest/gtest.h:10,
                 from ../../testing/gtest-support.h:8,
                 from ../../test/unittests/test-utils.h:20,
                 from ../../test/unittests/compiler/backend/instruction-selector-unittest.h:15,
                 from ../../test/unittests/compiler/x64/instruction-selector-x64-unittest.cc:9:
../../third_party/googletest/src/googletest/include/gtest/internal/gtest-param-util.h: In instantiation of ...
warning: '...' has a field '...' whose type uses the anonymous namespace [-Wsubobject-linkage]
```
这里的问题是 `InstructionSelectorChangeInt32ToInt64Test` 使用了一个在匿名命名空间中的类型，但由 `TEST_P` 生成的类不是。在这种情况下，它将仅在单个编译单元中使用，关于 subobject-linkage 的警告可以通过将 `TEST_P` 生成的类也放入匿名命名空间来避免。

要查看 gtest 宏展开后的内容，可以使用以下命令：
```console
$ g++ -I./deps/googletest/googletest/include -E test/parameterized_test.cc
```
```cpp
class SomeParamTest_Something_Test : public SomeParamTest { 
  public: 
    SomeParamTest_Something_Test() {} 
    virtual void TestBody(); 
  private: 
    static int AddToRegistry() { 
      ::testing::UnitTest::GetInstance()->parameterized_test_registry()
        .GetTestCasePatternHolder<SomeParamTest>( "SomeParamTest",
          ::testing::internal::CodeLocation( "test/parameterized_test.cc", 16))->AddTestPattern( "SomeParamTest", "Something",
          new ::testing::internal::TestMetaFactory< SomeParamTest_Something_Test>());
      return 0; 
    }
    static int gtest_registering_dummy_ __attribute__ ((unused));
    SomeParamTest_Something_Test(SomeParamTest_Something_Test const &) = delete;
    void operator=(SomeParamTest_Something_Test const &) = delete;
};
    int SomeParamTest_Something_Test::gtest_registering_dummy_ = SomeParamTest_Something_Test::AddToRegistry();

    void SomeParamTest_Something_Test::TestBody() {
      std::cout << "Something: param: " << GetParam() << '\n';
    }
```
