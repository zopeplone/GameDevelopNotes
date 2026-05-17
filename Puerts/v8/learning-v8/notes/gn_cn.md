### [GN](https://chromium.googlesource.com/chromium/src/+/master/tools/gn/docs/quick_start.md)
GN 代表 Generate Ninja，可执行文件 `gn` 是一个调用 `gn.py` 的 bash 脚本。

```console
$ tools/dev/v8gen.py --help
```

显示所有可执行文件：
```console
$ gn ls out/x64.release_gcc --type=executable
```
如果你想查看特定 target 的详细信息，可以使用 `desc`：
```console
$ gn desc out/x64.release_gcc/ //:mksnapshot
``` 
`//:mksnapshot` 是一个 label，其中 `//` 表示根目录，然后是 label 的名称。如果 label 在子目录中，该子目录应出现在 `:` 之前。
这个命令对于查看可执行文件中包含哪些文件非常有用。

当 gn 启动时，它会在当前目录中搜索 .gn 文件。V8 中的 .gn 文件指定了以下内容：
```
import("//build/dotfile_settings.gni")                                          
```
这从 build 目录（即 dotfile_settings.gni）中包含了一堆 gni 文件。

```console
buildconfig = "//build/config/BUILDCONFIG.gn"                                   
```
这是主 GN build 配置文件。

显示所有 shared_libraries：
```console
$ gn ls out/x64.release_gcc --type=shared_libraries
```

编辑 build 参数：
```console
$ vi out/x64.release_gcc/args.gn
$ gn args out/x64.release_gcc
```

这将打开一个编辑器，你可以在其中设置配置选项。我一直使用以下配置：
```console
v8_monolithic = true
use_custom_libcxx = false
v8_use_external_startup_data = false
is_debug = true
target_cpu = "x64"
v8_enable_backtrace = true
v8_enable_slow_dchecks = true
v8_optimized_debug = false
```

列出可用的 build 参数：
```console
$ gn args --list out/x64.release
```

列出所有可用的 targets：
```console
$ ninja -C out/x64.release/ -t targets all
```

构建：
```console
$ env CPATH=/usr/include ninja -C out/x64.release_gcc/
```

运行 quickchecks：
```
$ ./tools/run-tests.py --outdir=out/x64.release --quickcheck
```
你可以使用 `./tools-run-tests.py -h` 列出可以传递给 run-tests 的所有选项。
