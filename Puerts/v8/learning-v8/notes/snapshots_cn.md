### Snapshots
JavaScript 指定了大量 built-in 功能，每个 V8 context 都必须提供这些功能。例如，你可以运行 `Math.PI`，这在 JavaScript 控制台/repl 中是可以工作的。

global object 和所有 built-in 功能必须设置并初始化到 V8 heap 中。如果每次都必须这样做，这可能非常耗时并影响运行时性能。

这就是 `snapshot_blob.bin` 文件发挥作用的地方。
但这个 bin 文件是什么？
此 blob 是一个准备好的 snapshot，可以直接反序列化到 heap 中，以提供已初始化的 context。

当 V8 使用 `v8_use_external_startup_data` 构建时，构建过程将使用 `BUILD.gn` 中名为 `run_mksnapshot` 的模板创建 `snapshot_blob.bin` 文件，但如果为 false，它将生成一个名为 `snapshot.cc` 的文件。

#### mksnapshot
此可执行文件用于创建 snapshot，它启动一个 V8 Isolate，然后将运行时数据序列化为二进制格式。

有一个名为 `mksnapshot` 的可执行文件，定义在 `src/snapshot/mksnapshot.cc` 中。这可以在命令行上执行：
```console
$ mksnapshot --help
Usage: ./out/x64.release_gcc/mksnapshot [--startup-src=file] [--startup-blob=file] [--embedded-src=file] [--embedded-variant=label] [--target-arch=arch] [--target-os=os] [extras]
```
但奇怪的是使用说明从未被打印，取而代之的是正常的 V8/D8 选项。如果我们查看 `src/snapshot/snapshot.cc`，我们可以看到：
```cpp
  // 如果解析命令行 flags 时发生错误或设置了 help flag，则打印使用说明。
  int result = i::FlagList::SetFlagsFromCommandLine(&argc, argv, true, false);
  if (result > 0 || (argc > 3) || i::FLAG_help) {
    ::printf("Usage: %s --startup_src=... --startup_blob=... [extras]\n",
             argv[0]);
    i::FlagList::PrintHelp();
    return !i::FLAG_help;
  }
```
`SetFlagsFromCommandLine` 定义在 `src/flags/flags.cc` 中：
```cpp
int FlagList::SetFlagsFromCommandLine(int* argc, char** argv, bool remove_flags) {
  ...
  if (FLAG_help) {
    PrintHelp();
    exit(0);
  }
}
```
所以即使我们指定了 `--help`，snapshot 的使用说明字符串也不会被打印。作为建议，我创建了 https://chromium-review.googlesource.com/c/v8/v8/+/2290852 来解决这个问题。

通过这个更改，`--help` 将打印使用说明消息：
```console
$ ./out/x64.release_gcc/mksnapshot --help | head -n 1
Usage: ./out/x64.release_gcc/mksnapshot --startup_src=... --startup_blob=... [extras]
```

mksnapshot 所做的与普通 V8 应用程序的启动大致相同，它必须初始化 V8 本身（`src/snapshot/snapshot.cc`）：
```cpp
  v8::V8::InitializeICUDefaultLocation(argv[0]);
  std::unique_ptr<v8::Platform> platform = v8::platform::NewDefaultPlatform();
  v8::V8::InitializePlatform(platform.get());
  v8::V8::Initialize();
```
注意，这与我们在 [hello-world](../hello-world.cc) 或 [v8_test_fixture.h](../test/v8_test_fixture.h) 中所做的完全一样。

接下来，我们有：
```cpp
  SnapshotFileWriter snapshot_writer;
  snapshot_writer.SetSnapshotFile(i::FLAG_startup_src);
  snapshot_writer.SetStartupBlobFile(i::FLAG_startup_blob);
```
`i::FLAG_startup_src` 定义在 `src/flags/flag-definitions.h` 中：
```cpp
DEFINE_STRING(startup_src, nullptr,
              "Write V8 startup as C++ src. (mksnapshot only)")

#define DEFINE_STRING(nam, def, cmt) FLAG(STRING, const char*, nam, def, cmt)

#undef FLAG
#ifdef ENABLE_GDB_JIT_INTERFACE
#define FLAG FLAG_FULL
#else
#define FLAG FLAG_READONLY
#endif

#define FLAG_FULL(ftype, ctype, nam, def, cmt) \
  V8_EXPORT_PRIVATE extern ctype FLAG_##nam;

#define FLAG_READONLY(ftype, ctype, nam, def, cmt) \
  static constexpr ctype FLAG_##nam = def;
```
因此上述宏将被预处理器展开为：
```c
  V8_EXPORT_PRIVATE extern const char* FLAG_startup_src;
```

### V8 如何创建 snapshot
mksnapshot 可执行文件接受多个参数，其中之一是 target OS，它可以与 host OS 不同。target 的示例可以在 `BUILD.gn` 中找到，包括 `android`、`fuchsia`、`ios`、`linux`、`mac` 和 `win`。

`BUILD.gn` 中有一个名为 `run_mksnapshot` 的模板，展示了如何调用 `mksnapshot`。

如果在 `args.gn` 中设置了 `v8_use_external_startup_data`，选项 `--startup_blob` 将被传递给 mksnapshot：
```console
  if (v8_use_external_startup_data) {
      outputs += [ "$root_out_dir/snapshot_blob${suffix}.bin" ]
      data += [ "$root_out_dir/snapshot_blob${suffix}.bin" ]
      args += [
        "--startup_blob",
        rebase_path("$root_out_dir/snapshot_blob${suffix}.bin", root_build_dir),
      ]
```
然后这个二进制文件可以被反序列化。如果 `v8_use_external_startup_data` 未设置，则改为传递 `--startup_src`：
```
  } else {
      outputs += [ "$target_gen_dir/snapshot${suffix}.cc" ]
      args += [
        "--startup_src",
        rebase_path("$target_gen_dir/snapshot${suffix}.cc", root_build_dir),
      ]
    }
```
要查看构建时实际使用的内容，可以查看文件 `out/x64.release_gcc/toolchain.ninja`：
```console
rule ___run_mksnapshot_default___build_toolchain_linux_x64__rule
  command = python ../../tools/run.py ./mksnapshot --turbo_instruction_scheduling 
    --target_os=linux 
    --target_arch=x64 
    --embedded_src gen/embedded.S 
    --embedded_variant Default 
    --random-seed 314159265 
    --startup_src gen/snapshot.cc 
    --native-code-counters 
    --verify-heap
  description = ACTION //:run_mksnapshot_default(//build/toolchain/linux:x64)
  restat = 1
  pool = build_toolchain_action_pool
```

### startup-src
此选项指定将由 mksnapshot 生成的 C++ 源文件的路径，例如：
```console
$ ../v8_src/v8/out/x64.release_gcc/mksnapshot --startup_src=gen/example_snapshot.cc
```
之后我们可以检查 `gen/example_snapshot.cc`，它将包含名为 `blob_data` 的字节数组中的序列化 snapshot。
```cpp
#include "src/snapshot/snapshot.h"

namespace v8 {
namespace internal {

alignas(kPointerAlignment) static const byte blob_data[] = {
  ...
};
static const int blob_size = 133115;
static const v8::StartupData blob = { (const char*) blob_data, blob_size };
const v8::StartupData* Snapshot::DefaultSnapshotBlob() {
  return &blob;
}
```
此文件由 `MaybeWriteSnapshotFile` 生成。

### startup-blob
这也是可以传递给 `mksnapshot` 的命令行选项，它将在指定路径生成一个二进制文件。注意，这可以在 `startup-src` 之外额外指定，这是我之前没有意识到的：
```
$ ../v8_src/v8/out/x64.release_gcc/mksnapshot --startup_src=gen/example-snapshot.cc --startup-blob=gen/example-blob.bin
$ ls -l -h gen/example-blob.bin
-rw-rw-r--. 1 danielbevenius danielbevenius 130K Jul 14 09:52 gen/example-blob.bin
```
然后可以使用以下方式指定此二进制文件：
```cpp
  V8::InitializeExternalStartupDataFromFile("../gen/example-blob.bin");
```

`src/init/startup-data-util.cc` 包含此功能的实现，请注意它受宏保护：
```cpp
void InitializeExternalStartupDataFromFile(const char* snapshot_blob) {
#ifdef V8_USE_EXTERNAL_STARTUP_DATA
  LoadFromFile(snapshot_blob);
#endif  // V8_USE_EXTERNAL_STARTUP_DATA
}
```
因此我们需要确保已将 `v8_use_external_startup_data` 设置为 true。
