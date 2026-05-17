## EcmaScript Spec
为了加深我对 V8 代码库的理解，我发现阅读 [规范](https://tc39.es/ecma262/) 非常有帮助。这可以清楚地说明为什么函数和字段以特定方式命名，以及为什么它们执行某些操作。

### [[something]]
通常表示名为 `something` 的内部属性。

### @@
在规范中，well-known symbol 使用 `@@name` 形式的表示法来引用，其中 `name` 是以下值之一：
```
Spec Name               [[Description]]
* @@asyncIterator       Symbol.asyncIterator
* @@hasInstance         Symbol.hasInstance
* @@isConcatSpreadable  Symbol.isConcatSpreadable
* @@iterator            Symbol.iterator
* @@match               Symbol.match
* @@matchAll            Symbol.matchAll
* @@replace             Symbol.replace
* @@search              Symbol.search
* @@speices             Symbol.species
* @@split               Symbol.split
* @@toPrimitive         Symbol.toPrimitive
* @@toStringTag         Symbol.toStringTag
```
所以你可能会看到类似 [@@toPrimitive] 的表示，一个对象会有一个以 Symbol.toPrimitive 为键的函数，可用于访问该函数。

### Record
规范中的 Record 类似于 C 语言中的 struct，其中每个成员称为一个 field。

### Completion Record
Completion Record 是一种用作返回值的 Record，可以具有以下三种可能的字段之一：
```
[[Type]]（normal、return、throw、break 或 continue）
```
如果类型是 normal、return 或 throw，则 CompletionRecord 可以具有：
```
[[Value]]，即返回/抛出的值。
```
```
[[Target]] 如果类型是 break 或 continue，则可以选择性地具有 [[Target]]。
```

例如：
```js
function something() {
  if (bla) {
    return CompletionRecord({type: "normal", value: "something"});
  } else {
    return CompletionRecord({type: "throw", value: "error"});
  }
}
const result = something();
```
因此，规范中的函数会返回一个 CompletionRecord，规范必须描述每种情况下要执行的操作。

### ReturnIfAbrupt
如果在上面的示例中它是 abrupt（throw），则返回该值。规范编写者可以使用以下方式代替文本来表达：
```
ReturnIfAbrupt(something())
```
可以写成：
```
? something()
```
ReturnIfAbrupt 意味着如果传入的值是 abrupt completion，则返回该值；否则，如果该值是 Completion Record，则将值设置为 value.[[Value]]。

```js
function something() {
  return CompletionRecord({type: "normal", value: "something"});
}
const result = something();
```
```
Let result be !something()
```
这意味着 something() 将"永不"返回 abrupt completion。

规范中的每个 abstract operation 都隐式地返回一个 completion record。

### Sets
你可以在规范中看到 sets 的使用，例如 `<< one, two >>`，表示具有两个元素的 set。

### Builtin Objects
在规范中，可以看到对象以 `%Array%` 的形式引用，这指的是 builtin 对象。这些对象列在 [well-known-intrinsic-objects](https://tc39.es/ecma262/#sec-well-known-intrinsic-objects) 中。

### Realms
所有 JS 代码在求值之前必须与一个 realm 关联，realm 由 builtin/intrinsic 对象和一个 global environment 组成（[链接](https://tc39.es/ecma262/#sec-code-realms)）。
这可以是浏览器中的 window/frame/tab，另一个例子可以是 web worker 或 service worker。一个 realm 中的代码可能无法与另一个 realm 中的代码交互。例如，一个 symbol 可能只在一个 realm 中可用。
TODO: 添加更多示例并澄清这一点。
