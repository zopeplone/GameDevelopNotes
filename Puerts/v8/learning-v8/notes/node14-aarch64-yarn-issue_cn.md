# ARM64 Yarn Node 14.15.4 问题
本文档包含与在 aarch64 系统上运行 Node.js 14.15.4 和 Yarn 时看到的崩溃相关的笔记。

## 构建问题
在启用 debug 构建 Node 时，mksnapshot 生成以下错误：
```console
$ ./configure --debug && make -j8
 ...
 ln -fs out/Release/node node; fi
  touch 768c82b5c8db30aa6ca639ca252071b77b55f399.intermediate
  LD_LIBRARY_PATH=/home/sxa/node/out/Debug/lib.host:/home/sxa/node/out/Debug/lib.target:$LD_LIBRARY_PATH; export LD_LIBRARY_PATH; cd ../tools/v8_gypfiles; mkdir -p /home/sxa/node/out/Debug/obj.target/v8_snapshot/geni; "/home/sxa/node/out/Debug/mksnapshot" --turbo_instruction_scheduling "--target_os=linux" "--target_arch=arm64" --startup_src "/home/sxa/node/out/Debug/obj.target/v8_snapshot/geni/snapshot.cc" --embedded_variant Default --embedded_src "/home/sxa/node/out/Debug/obj.target/v8_snapshot/geni/embedded.S" --no-native-code-counters


#
# Fatal error in ../deps/v8/src/heap/memory-chunk.cc, line 50
# Debug check failed: kMaxRegularHeapObjectSize <= memory (131072 vs. 65536).
#
#
#
#FailureMessage Object: 0x3ffc32ff240
==== C stack trace ===============================

    /home/sxa/node/out/Debug/mksnapshot(v8::base::debug::StackTrace::StackTrace()+0x18) [0x209e480]
/bin/sh: line 1: 29000 Trace/breakpoint trap   "/home/sxa/node/out/Debug/mksnapshot" ...
```
这是一个 debug 检查，在进行 release 构建时不会发生。以下 [issue](https://bugs.chromium.org/p/v8/issues/detail?id=10808) 包含与此相同错误的报告。

注意，上述系统的 pagesize 是：
```console
$ getconf PAGESIZE
65536
```

## 复现
```console
$ uname -m
aarch64

$ curl https://nodejs.org/dist/v14.15.4/node-v14.15.4-linux-arm64.tar.gz | tar xpfz -
$ cd node-v14.15.4-linux-arm64/                                                 
$ export PATH=$PWD/bin:$PATH                                                    

$ node --version                                                                
v14.15.4                                                                        

$ npm --version                                                                 
6.14.10                                                                         

$ mkdir tmp && cd tmp                                                           
$ npm install -g yarn                                                           
$ npm init                                                                      
$ yarn add lodash                                                               
yarn add v1.22.10                                                               
info No lockfile found.                                                         
[1/4] Resolving packages...                                                     
[2/4] Fetching packages...                                                      
[-] 0/1Segmentation fault (core dumped)
```

### 排查
为了能够进行 debug 构建，我们需要应用 [补丁](https://github.com/v8/v8/commit/f4376ec801e1dedfde5309b2cb50ed70f052b7e4.patch)，以便构建能够通过（当前在 mksnapshot 上失败，如上所述）。然后我们应该能够在调试器中运行 yarn 来查看问题（如果问题不同，可能只是 release 构建中此问题的表现形式）。

上述补丁不能直接应用，需要使用此 [补丁](aarch64-node-14.15.4-pagesize.patch) 手动应用到 Node.js 14.15.4。

应用该补丁并重试上述 yarn 命令后：
```console
$ cd danbev/test
$ export PATH=/home/sxx/node:$PATH
$ /home/sxa/node/deps/npm/bin/npm-cli.js init
$ /home/sxa/node/deps/npm/bin/npm-cli.js install yarn
$ ./node_modules/yarn/bin/yarn add lodash 
yarn add v1.22.10
warning ../../package.json: No license field
[1/4] Resolving packages...
[2/4] Fetching packages...
[3/4] Linking dependencies...
[4/4] Building fresh packages...
success Saved 1 new dependency.
info Direct dependencies
└─ lodash@4.17.20
info All dependencies
└─ lodash@4.17.20
Done in 4.19s.
```

[关于 V8 中 page sizes 的笔记](./heap_cn.md#pagesize-in-v8) 可能有助于理解此问题。

14.15.4 中使用的 V8 版本是：
```console
[sxa@147 node]$ ./node -p process.versions
{
  node: '14.15.4',
  v8: '8.4.371.19-node.17',
}
```

这个 [pull request](https://github.com/nodejs/node/pull/37225) 是针对此问题提交的。
