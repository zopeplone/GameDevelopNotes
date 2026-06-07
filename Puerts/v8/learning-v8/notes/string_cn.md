### String 类型
V8 中有多种不同的 String 类型，针对各种情况进行了优化。
如果我们查看 `src/objects/objects.h`，我们可以看到对象层次结构：
```
    Object
      SMI
      HeapObject    // 在 heap 上分配的每个对象实例的父类。
        ...
        Name
          String
            SeqString
              SeqOneByteString
              SeqTwoByteString
            SlicedString
            ConsString
            ThinString
            ExternalString
              ExternalOneByteString
              ExternalTwoByteString
            InternalizedString
              SeqInternalizedString
                SeqOneByteInternalizedString
                SeqTwoByteInternalizedString
              ConsInternalizedString
              ExternalInternalizedString
                ExternalOneByteInternalizedString
                ExternalTwoByteInternalizedString
```

注意，`v8::String` 在 `include/v8.h` 中声明。

`Name` 继承自 HeapObject，任何可以用作属性名的东西都应该继承 Name。

查看 `include/v8.h` 中的声明，我们找到以下内容：
```cpp
    int GetIdentityHash();
    static Name* Cast(Value* obj)
```

### Unicode
本节包含一些关于 Unicode 的笔记，在阅读代码和理解某些与 string 相关的函数功能时，记住这些是很有用的。

#### Abstract characters
Unicode 中的 abstract character 有一个名称，如 `LATIN SMALL LETTER A`。

#### Code point
code point 是与 abstract character 关联的数字，例如 U+0061（U+<hex>，其中 U 代表 Unicode）。

#### Planes
plane 是从 U+n0000 到 U+nFFFF 的 65536 个连续 code point 的范围。共有 17 个等分组：
```
Plane 0: U+0000 -> U+FFFF           Basic Multilingual Plane (BMP)
Plane 1: U+10000 -> U+1FFFF         Supplementary Multilingual Plane
Plane 2: U+20000 -> U+2FFFF         Supplementary Ideographic Plane
Plane 3: U+30000 -> U+3FFFF
...
Plane 16: U+100000 -> U+10FFFF      Supplementary Private Use Area B.
```
BMP 包含了编程时使用的大部分字符，这些字符可以有 4 个十六进制数字。

#### Code units
不要与 code point 混淆，code point 只是用于以 U+<hex> 格式查找 abstract character 的数字。计算机内存不处理 code point 或 abstract character，而是处理 code unit（即位序列）。
你可以将 code point 通过一个函数转换，将其翻译为 code unit。这个过程称为 character encoding。有不同的编码方式，JavaScript 使用 UTF-16。

#### 16 位 Unicode 转换格式 (UTF-16)
来自 BMP 的 code point 被转换/编码为单个 16 位的 code unit。
来自其他 planes 的 code point 被转换/编码为两个 code unit，每个大小为 16 位。

```
Abstract Character        Code point       Code unit
LATIN SMALL LETTER A      U+0061           0x0061
LATIN SMALL LETTER B      U+0062           0x0062
...
```

#### Surrogate pairs
要表示瑞典语字符 'å'，我们需要两个各 16 位的 code unit：
```console
$ node -p "console.log('\u0061\u030A')"
å
```
这对 code unit 表示一个单一的 abstract character。第一个称为 high-surrogate code unit，第二个称为 low-surrogate code unit。

对于 'å'，第二个 code unit 称为 combining mark，因为它修饰了前面的字符。

#### String
String 继承自 `Name`，具有长度和内容。内容可以由 1 或 2 字节的字符组成。
查看 `include/v8.h` 中的声明，我们找到以下内容：
```cpp
    enum Encoding {
      UNKNOWN_ENCODING = 0x1,
      TWO_BYTE_ENCODING = 0x0,
      ONE_BYTE_ENCODING = 0x8
    };

    int Length() const;
    int Utf8Length const;
    bool IsOneByte() const;
```
示例用法见 [test/string_test.cc](./test/string_test.cc)。
在查看函数时，我看到了一个从 String 返回实际字节的函数。你可以使用以下方式以 utf8 格式获取：
```cpp
    String::Utf8Value print_value(joined);
    std::cout << *print_value << '\n';
```
所以这是 `include/v8.h` 中唯一的 string 类，但还有更多我们在上面看到的实现。它们用于各种情况，例如索引、连接和切片。

#### SeqString
表示一个字符序列，其中（字符）长度为一或两个字节。

#### ConsString
这些是使用以下方式构建的字符串：

    const str = "one" + "two";

这将表示为：
```
         +--------------+
         |              | 
   [str|one|two]     [one|...]   [two|...]
             |                       |
             +-----------------------+
```
所以我们可以看到 str 中的 one 和 two 是指向现有字符串的指针。

### NewFromUtf8Literal
此函数在 b097a8e5de7 中引入。

创建 `Local<String>` 的常规方式是使用：
```cpp
  Local<String> str = String::NewFromUtf8(isolate_, "åäö").ToLocalChecked();    
```
现在，`String::NewFromUtf8` 如下所示：
```cpp
MaybeLocal<String> String::NewFromUtf8(Isolate* isolate, const char* data,           
                                       NewStringType type, int length) {        
  NEW_STRING(isolate, String, NewFromUtf8, char, data, type, length);           
  return result;                                                                
}
```
宏 `NEW_STRING`（可以在 `src/api/api.cc` 中找到）。
我们可以使用以下命令查看展开后的宏：
```console
$ g++ -I./out/x64.release_gcc/gen -I./include -I. -E src/api/api.cc > output
```
```cpp
MaybeLocal<String> String::NewFromUtf8(Isolate* isolate, const char* data,      
                                       NewStringType type, int length) {        
  MaybeLocal<String> result;
  if (length == 0) {
    result = String::Empty(isolate);
  } else if (length > i::String::kMaxLength) {
    result = MaybeLocal<String>();
  } else {
    i::Isolate* i_isolate = reinterpret_cast<internal::Isolate*>(isolate);
    i::VMState<v8::OTHER> __state__((i_isolate));;
    i::RuntimeCallTimerScope _runtime_timer( i_isolate, i::RuntimeCallCounterId::kAPI_String_NewFromUtf8);
    do {
      auto&& logger = (i_isolate)->logger();
      if (logger->is_logging())
        logger->ApiEntryCall("v8::" "String" "::" "NewFromUtf8");
    } while (false);
    if (length < 0)
      length = StringLength(data);
     i::Handle<i::String> handle_result = NewString(i_isolate->factory(), type, i::Vector<const char>(data, length)) .ToHandleChecked();
     result = Utils::ToLocal(handle_result);
  };
  return result;                                                                
}
```
当使用 string literal 时，有许多检查是不需要的，而有些可以在编译时检查，例如最大长度。
```cpp
  template <int N>                                                              
  static V8_WARN_UNUSED_RESULT Local<String> NewFromUtf8Literal(                
      Isolate* isolate, const char (&literal)[N],                                    
      NewStringType type = NewStringType::kNormal) {                                 
    static_assert(N <= kMaxLength, "String is too long");                       
    return NewFromUtf8Literal(isolate, literal, type, N - 1);                        
  }                                                                                 
```
