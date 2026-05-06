https://www.nowcoder.com/discuss/353157797267316736

# 1. 移动构造函数与拷贝构造函数对比
> 拷贝构造函数用于从已有对象复制数据来初始化新对象，通常会进行深拷贝，例如重新分配内存并复制资源。
> 移动构造函数是 C++11 引入的，它接受右值引用参数，通过转移资源所有权来初始化对象，而不是复制资源。
> 这样可以避免昂贵的内存分配和数据复制，大幅提高性能。STL 容器如 vector、string 都大量使用移动语义优化性能。
## 一、拷贝构造函数（Copy Constructor）

### 定义

拷贝构造函数用于 **用一个已有对象创建新对象，并复制其内容**。
```cpp
class A {  
public:  
    A(const A& other) {  
        // 拷贝other的数据  
    }  
};
```
`const A&`
表示 **引用传递避免再次拷贝**。

---

### 示例
```cpp
class Buffer {  
public:  
    int* data;  
  
    Buffer(int size) {  
        data = new int[size];  
    }  
  
    // 拷贝构造  
    Buffer(const Buffer& other) {  
        data = new int[10];  
        memcpy(data, other.data, sizeof(int) * 10);  
    }  
};
```


调用：
Buffer a(10);  
Buffer b = a;   // 调用拷贝构造
此时：
a.data -> 内存A  
b.data -> 内存B
**复制了一份新的内存**

---

## 二、移动构造函数（Move Constructor）

C++11 引入。

### 定义

移动构造函数用于 **转移资源所有权，而不是复制资源**。
```cpp
class A {  
public:  
    A(A&& other) {  
        // 转移资源  
    }  
};
```

参数：`A&&` ,这是 **右值引用**。

---

### 示例
```cpp
class Buffer {  
public:  
    int* data;  
  
    Buffer(int size) {  
        data = new int[size];  
    }  
  
    // 移动构造  
    Buffer(Buffer&& other) {  
        data = other.data;   // 直接拿走资源  
        other.data = nullptr;  
    }  
};
```
调用：
Buffer a(10);  
Buffer b = std::move(a);   // 调用移动构造
此时：
a.data -> nullptr  
b.data -> 原来的内存
**资源被转移了，没有重新分配内存。**

---

## 三、核心区别

|      | 拷贝构造       | 移动构造     |
| ---- | ---------- | -------- |
| 参数   | `const T&` | `T&&`    |
| 语义   | 复制资源       | 转移资源     |
| 性能   | 较慢         | 非常快      |
| 内存   | 新申请        | 不申请      |
| 对象状态 | 原对象不变      | 原对象被“掏空” |

---

## 四、性能差异（非常重要）

举例：
```cpp
std::vector<int> a(1000000);  
std::vector<int> b = a;
```

- 拷贝构造：
分配1000000内存  
复制1000000数据
时间复杂度：O(n)
- 移动构造：
```cpp
std::vector<int> b = std::move(a);
```
发生：指针交换
时间复杂度：O(1)

所以 **STL 容器大量使用移动语义**。


# 2. 空类

## 空类有哪些函数？
例如：
`class A {};`
即使你什么都没写，**编译器也会自动生成 6 个函数**（C++11 以后）。
1.  默认构造函数 `A();`
2. 析构函数 `~A();`
3. 拷贝构造函数 `A(const A&);`
4. 拷贝赋值运算符 `A& operator=(const A&);`
5. 移动构造函数（C++11）`A(A&&);`
6. 移动赋值运算符（C++11）`A& operator=(A&&);`
## 空类的大小是多少?
`class A {};` `sizeof(A)` 
结果为 1
**C++要求每个对象必须有唯一地址**
例如：
`A a;`  
`A b;`
如果 `sizeof(A)==0`
那么`&a == &b`
这违反 C++ 对象模型。 所以编译器会**插入一个字节**
逻辑上类似：
```cpp
class A {  
    char dummy;  
};
```
因此：`sizeof(A) = 1`

- 如果有虚函数会怎样
```cpp
class A {  
    virtual void func();  
};
```
此时对象会有**vptr**
在 64 位系统：`sizeof(A) = 8` ,因为：虚函数表指针 vptr存在于对象内

## EBO
**EBO（Empty Base Optimization）** 是 C++ 的一种 **编译器优化**：

> **当一个类继承自空类时，编译器可以不为这个空基类分配额外空间。**

也就是说：**空基类不会增加派生类对象的大小。**
### 一、为什么需要 EBO

先看一个空类：

class Empty {};

我们知道：

sizeof(Empty) == 1

因为 **C++ 要保证每个对象有唯一地址**。

---

如果我们把它作为成员：

class Test {  
    Empty e;  
    int x;  
};

对象布局可能是：

e : 1 byte  
padding  
x : 4 bytes

结果：

sizeof(Test) = 8

因为 **成员对象必须占空间**。

---

### 二、EBO 的情况

如果是 **继承空类**：

class Empty {};  
  
class Test : public Empty {  
    int x;  
};

对象布局：

x : 4 bytes

结果：

sizeof(Test) == 4

空基类 **不占空间**。

这就是 **EBO**。

---

### 三、为什么继承可以优化

原因是：

**基类子对象可以和派生类共享地址。**

例如：

Test对象地址 = Empty子对象地址

而成员对象不行，因为：

成员必须有独立地址

---

### 四、验证代码
```cpp
#include <iostream>  
using namespace std;  
  
class Empty {};  
  
class A {  
    Empty e;  
    int x;  
};  
  
class B : public Empty {  
    int x;  
};  
  
int main() {  
    cout << sizeof(A) << endl;  
    cout << sizeof(B) << endl;  
}
/**典型输出：
8  
4
/**
```
---
### 五、STL 为什么大量使用 EBO

一个经典例子：

**allocator**

例如：

```cpp
std::vector<int>

内部结构类似：

template<class T, class Allocator>  
class vector : private Allocator {  
    T* start;  
    T* finish;  
};
```
如果 allocator 是空类：

sizeof(vector)

不会增加。

否则每个容器都会 **多 1 字节甚至更多**。

所以 STL 用 **继承而不是成员** 来存储 allocator。


### 六、EBO 的限制

EBO **不是绝对能发生**。
1. 多个相同空基类

```cpp
class Empty {};  
  
class Test : public Empty, public Empty {};
```

不允许，因为地址会冲突。

---

2. 成员对象不会触发
```cpp
class Test {  
    Empty e;  
};
```
还是：sizeof = 1
3. 虚继承也不行
```cpp
class Test : virtual public Empty {};
```
因为需要 **虚基类指针**。



# 3. struct与class的区别

在 C++ 中 **`struct` 和 `class` 本质上几乎一样**，唯一的核心区别只有 **两个默认规则**

| 区别     | struct | class   |
| ------ | ------ | ------- |
| 默认访问权限 | public | private |
| 默认继承方式 | public | private |
`class`继承`struct` 继承方式是什么？

```cpp
struct A {};  
class B : A {};
```
答案：private
因为 **class 默认继承是 private**。

# 内存分区

# 内存对齐

# new/delete与malloc/free的区别


# 虚函数实现动态多态的原理、虚函数与纯虚函数的区别

# 继承时，父类的析构函数是否为虚函数？构造函数能不能为虚函数？为什么？

# 静态多态：重写、重载、模板

# static关键字：修饰局部变量、全局变量、类中成员变量、类中成员函数

# const关键字：修饰变量、指针、类对象、类中成员函数


# extern

# 四种类型转换：static_cast、dynamic_cast、const_cast、reinterpret_cast

# 右值引用
