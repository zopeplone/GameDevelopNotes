### Arrays
Array 的声明可以在 `include/v8.h` 中找到，它继承自 Object。它有一个 `Length()` 方法和用于创建新实例的函数，但除此之外，它只有从 Object 继承的方法。

对象具有映射到值的属性（properties），而数组具有映射到元素的索引（indices）。V8 以不同的方式存储它们，以便能够优化某些操作。

示例见 [arrays_test.cc](../test/arrays_test.cc)。

### Holey Array
holey array 是包含空洞的数组，这可能发生在数组被创建为特定大小但在创建时未赋予任何值，或者某个条目已从数组中删除的情况下。当存在空洞时，V8 必须查找 prototype chain 以查看该值是否可能存在于那里。这需要时间，如果 V8 知道没有空洞，就可以进行优化。

即使 holey array 包含正在查找的值，V8 也必须执行边界检查（bounds check），然后调用 `hasOwnProperty`。

### Packed Array
packed array 是不包含任何空洞的数组。这意味着 V8 不需要查找 prototype chain 来获取值，因为正在查找的值要么在数组中，要么不在，因此 V8 可以对正在查找的索引进行 bounds check 并返回该处的值。

### ElementsKinds
`src/objects/elements-kind.h`
