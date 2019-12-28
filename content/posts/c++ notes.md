---
title: "C++ notes"
date: 2011-12-23T13:05:31Z
draft: true
---


# C++ notes

## 输出预处理宏

```sh
cpp -dM /dev/null
```

## 输出头文件标准搜索路径

```sh
cpp -v -o /dev/null /dev/null
```

## 整型大小

在32位机器上

* int 4
* long 4
* long long 8

在64位机器上

* int 4
* long 8
* long long 8

## literal

### character literal

* auto c0 =   'A'; // char
* auto c1 =  L'A'; // wchar_t
* auto c2 =  u'A'; // char16_t
* auto c3 =  U'A'; // char32_t

### string literal

* auto s0 =   "hello"; // const char *
* auto s1 =  L"hello"; // const wchar_t *
* auto s2 =  u"hello"; // const char16_t *, encoded as UTF-16
* auto s3 =  U"hello"; // const char32_t *, encoded as UTF-32
* auto s4 = u8"hello"; // const char *, encoded as UTF-8

### integer literal

* auto i0 = 1;          // int
* auto i1 = 1L;         // long
* auto i2 = 1LL;        // long long
* auto i3 = 1U;         // unsigned int
* auto i4 = 1UL;        // unsigned long
* auto i5 = 1ULL;       // unsigned long long

### floating point literal

* auto f0 = 1.0;        // double
* auto f1 = 1.0F;       // float
* auto f2 = 1.0L;       // long double

## const & constexpr

### const

```cpp
// const.h
const int BUF_SIZE = 1024;
```

`const`默认是`static`的，所以也可以在`const`前写`static`

```cpp
// const.h
static const int BUF_SIZE = 1024;
```

因为是`static`所以每个引用的源文件都拥有自己独立的`const`变量，不会产生多重定义的错误

如果我们想多个源文件共享同一个`const`定义，则需要在头文件中用`extern`显式声明，并在实现文件中定义

```cpp
// const.h
extern const int BUF_SIZE;
```

```cpp
// const.cpp
#include "const.h" // 这里要包含对BUF_SIZE的声明
const int BUF_SIZE = 1024;
```

### top level const

```cpp
int i = 1;
int *const p = &i;
```

`p`本身是常量，`p`指向的对象是变量

### low level const

```cpp
int i = 1;
const int *p = &i;
```

`p`本身是变量，`p`指向的对象是变量，但是不能通过`p`修改指向的变量

### constexpr

在编译时求值的表达式称为常量表达式

```cpp
constexpr int BUF_SIZE = 1024;
```

```cpp
const int *p = nullptr;         // p is a pointer to a const int
constexpr int *q = nullptr;     // q is a const pointer to int
```

constexpr 隐含 `top lever const`

在这里，`p`是变量，`q`是常量

## 类型别名

方法一

```cpp
typedef const char *CString;
```

方法二

```cpp
using CString = const char *
```

## decltype

`decltype(expr)` 返回表达式的类型

如果表达式是个简单变量，那么返回变量的类型\
否则如果表达式是

* 左值，返回左值类型引用
* 右值，返回右值类型

举例

```cpp
int i = 42;
int *p = &i;
int &r = i;

declexpr(i) a;          // 表达式是简单变量，a是int类型
declexpr(*p) b;         // 表达式是左值，b是int &类型
declexpr(r+1) c;        // 表达式是右值，c是int类型
decltype(i=1) d;        // 表达式是左值，d是int &类型
declexpr((i)) e;        // 表达式是左值，e是int &类型
```

注意最后一个例子，`(i)`不算简单变量，并且是左值，所以`e`是`int &`类型

## in-class initializer

类内初始化有一点需要注意的

```cpp
struct foo {
        int a = 1;      // ok
        int b{1};       // ok
        int c = {1};    // ok
        int e(1);       // error, can not use direct initialization
};
```

作为对比，列出类外变量初始化

```cpp
int main() {
        int a = 1;      // ok
        int b{1};       // ok
        int c = {1};    // ok
        int e(1);       // ok
}
```

区别只有最后一种情况，为了统一，最好在任何情况下都避免最后一种写法

## direct and copy initialization

```cpp
string s1 = "hello"; // copy initialization
string s2("hello");  // direct initialization
```

`copy initialization` 只能有一个初值

## default-, zero- and value-initialization

* zero-initialization 零初始化
* default-initialization 如果没有定义默认构造函数，则不初始化，否则调用默认构造函数
* value-initialization 如果没有定义默认构造函数，则零初始化，否则调用默认构造函数

[参考stackoverflow](https://stackoverflow.com/questions/620137/do-the-parentheses-after-the-type-name-make-a-difference-with-new/620402#620402)

上面的参考链接，重点看C++2003的带括号版本

## list initialization

TODO

When we use curly braces, {...}, we’re saying that, if possible, we want to list initialize the object.
If list initialization isn’t possible, the compiler looks for other ways to initialize the object from the given values.

## division and remainder

* / 如果除数和被除数的符号相同，则商是正数，否则为负数
* % 余数符号与被除数相同

求余公式

`(m / n) * n + m % n` 运算结果等于 `m`

## Array to Pointer Conversions

除了下列情况外，大多数情况下，数组名会自动转换为指针:

* decltype
* sizeof
* typeid
* address-of(&)
* initialize a reference to an array

## 保证求值顺序的4个操作符

* &&
* ||
* ? :
* ,

## 不要定义 top-lever const 形参

```cpp
void func(int i) {}
void func(const int i) {} // error: redefine
```

## Use Reference to const When Possible

## 管理指针参数的三种技术

1. Using a Marker to Specify the Extent of an Array
2. Explicitly Passing a Size Parameter
3. Using the Standard Library Conventions

## Names do not overload across scopes

usually it's a bad idea to declare functions at local scope

In C++, name lookup happens before type checking.

## 避免使用函数默认参数

## Put inline and constexpr Functions in Header Files
