## 介绍
V8 基本上由 heap 的内存管理和 execution stack 组成（非常简化，但有助于说明我的观点）。像 callback queue、event loop 以及 WebAPIs（DOM、ajax、setTimeout 等）之类的东西存在于 Chrome 中，或者在 Node 的情况下，APIs 是 Node.js APIs：
```
+------------------------------------------------------------------------------------------+
| Google Chrome                                                                            |
|                                                                                          |
| +----------------------------------------+          +------------------------------+     |
| | Google V8                              |          |            WebAPIs           |     |
| | +-------------+ +---------------+      |          |                              |     |
| | |    Heap     | |     Stack     |      |          |                              |     |
| | |             | |               |      |          |                              |     |
| | |             | |               |      |          |                              |     |
| | |             | |               |      |          |                              |     |
| | |             | |               |      |          |                              |     |
| | |             | |               |      |          |                              |     |
| | +-------------+ +---------------+      |          |                              |     |
| |                                        |          |                              |     |
| +----------------------------------------+          +------------------------------+     |
|                                                                                          |
|                                                                                          |
| +---------------------+     +---------------------------------------+                    |
| |     Event loop      |     |          Task/Callback queue          |                    |
| |                     |     |                                       |                    |
| +---------------------+     +---------------------------------------+                    |
|                             +---------------------------------------+                    |
|                             |          Microtask queue              |                    |
|                             |                                       |                    |
|                             +---------------------------------------+                    |
|                                                                                          |
|                                                                                          |
+------------------------------------------------------------------------------------------+
```
execution stack 是一个 frame pointer 的栈。对于每个被调用的函数，该函数将被推入 stack。当该函数返回时，它将被移除。如果该函数调用其他函数，这些函数也将被推入 stack。当它们全部返回后，执行可以从返回点继续。如果其中一个函数执行了一个耗时的操作，在它完成之前不会有任何进展，因为完成的唯一方式是函数返回并从 stack 中弹出。这就是单线程编程语言中发生的情况。

以上描述的是同步函数，那异步函数呢？
以调用 setTimeout 为例，setTimeout 函数将被推入 call stack 并执行。这就是 callback queue 和 event loop 发挥作用的地方。setTimeout 函数可以将函数添加到 callback queue。当 call stack 为空时，此队列将由 event loop 处理。

### Task
task 是可以通过将其放入 callback queue 来调度的函数。这由 `setTimeout` 和 `setInterval` 等 WebAPIs 完成。
当 event loop 开始执行 tasks 时，它将运行当前 task queue 中的所有 tasks。由 WebAPI 函数调用调度的任何新 tasks 只会被推入队列，但要到 event loop 的下一次迭代才会被执行。

当 execution stack 为空时，microtask queue 中的所有 tasks 将被运行，如果这些 tasks 中有任何一个向 microtask queue 添加了 tasks，这些 tasks 也将被运行，这与 task queue 处理这种情况的方式不同。

在 Node.js 中，`setTimeout` 和 `setInterval`...

### Micro task
micro task 是在当前函数运行完毕后、在 call stack 上当前的所有其他函数执行完毕后执行的函数。

Microtasks 内部信息可以在 [microtasks](./microtasks_cn.md) 中找到。

#### Microtask queue
当一个 promise 被创建时，它会立即执行，如果它已被 resolve，你可以对其调用 `then`。
