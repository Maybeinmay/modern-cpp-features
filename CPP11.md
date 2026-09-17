# C++11

## 概述
本文中的许多描述和示例来自各种资源（参见 [致谢](#致谢) 部分），并由我用自己的话进行了总结。

C++11 包含以下新的语言特性：
- [移动语义](#移动语义)
- [可变参数模板](#可变参数模板)
- [右值引用](#右值引用)
- [转发引用](#转发引用)
- [初始化器列表](#初始化器列表)
- [静态断言](#静态断言)
- [auto](#auto)
- [lambda 表达式](#lambda-表达式)
- [decltype](#decltype)
- [类型别名](#类型别名)
- [nullptr](#nullptr)
- [强类型枚举](#强类型枚举)
- [属性](#属性)
- [constexpr](#constexpr)
- [委托构造函数](#委托构造函数)
- [用户定义字面量](#用户定义字面量)
- [显式虚函数覆盖](#显式虚函数覆盖)
- [final 说明符](#final-说明符)
- [默认函数](#默认函数)
- [删除函数](#删除函数)
- [基于范围的 for 循环](#基于范围的-for-循环)
- [用于移动语义的特殊成员函数](#用于移动语义的特殊成员函数)
- [转换构造函数](#转换构造函数)
- [显式转换函数](#显式转换函数)
- [内联命名空间](#内联命名空间)
- [非静态数据成员初始化器](#非静态数据成员初始化器)
- [右尖括号](#右尖括号)
- [带引用限定的成员函数](#带引用限定的成员函数)
- [尾置返回类型](#尾置返回类型)
- [noexcept 说明符](#noexcept-说明符)
- [char32_t 和 char16_t](#char32_t-和-char16_t)
- [原始字符串字面量](#原始字符串字面量)

C++11 包含以下新的库特性：
- [std::move](#stdmove)
- [std::forward](#stdforward)
- [std::thread](#stdthread)
- [std::to_string](#stdto_string)
- [类型萃取](#类型萃取)
- [智能指针](#智能指针)
- [std::chrono](#stdchrono)
- [元组](#元组)
- [std::tie](#stdtie)
- [std::array](#stdarray)
- [无序容器](#无序容器)
- [std::make_shared](#stdmake_shared)
- [std::ref](#stdref)
- [内存模型](#内存模型)
- [std::async](#stdasync)
- [std::begin/end](#stdbeginend)

## C++11 语言特性

### 移动语义
移动一个对象意味着把它所管理的某些资源的所有权转移到另一个对象。

移动语义的第一个好处是性能优化。当一个对象即将到达其生命周期的终点时（要么因为它是个临时对象，要么因为显式调用了 `std::move`），移动通常是转移资源更廉价的方式。例如，移动一个 `std::vector` 只是把一些指针和内部状态拷贝到新的 vector 中——而复制则涉及复制 vector 中包含的每一个元素，如果旧的 vector 很快就会被销毁，这样做既昂贵又没有必要。

移动还使得像 `std::unique_ptr` 这样的不可复制类型（参见 [智能指针](#智能指针)）能够在语言层面保证同一时刻只有一个实例在管理某个资源，同时还能在作用域之间转移该实例。

参见：[右值引用](#右值引用)、[用于移动语义的特殊成员函数](#用于移动语义的特殊成员函数)、[`std::move`](#stdmove)、[`std::forward`](#stdforward)、[`转发引用`](#转发引用)。

### 右值引用
C++11 引入了一种新的引用，称为*右值引用*。对 `T` 的右值引用（`T` 是非模板类型参数，例如 `int` 或用户定义类型）使用语法 `T&&` 创建。右值引用只能绑定到右值。

左值和右值的类型推导：
```c++
int x = 0; // `x` is an lvalue of type `int`
int& xl = x; // `xl` is an lvalue of type `int&`
int&& xr = x; // compiler error -- `x` is an lvalue
int&& xr2 = 0; // `xr2` is an lvalue of type `int&&` -- binds to the rvalue temporary, `0`

void f(int& x) {}
void f(int&& x) {}

f(x);  // calls f(int&)
f(xl); // calls f(int&)
f(3);  // calls f(int&&)
f(std::move(x)); // calls f(int&&)

f(xr2);           // calls f(int&)
f(std::move(xr2)); // calls f(int&& x)
```

参见：[`std::move`](#stdmove)、[`std::forward`](#stdforward)、[`转发引用`](#转发引用)。

### 转发引用
也被（非正式地）称为*万能引用*。转发引用使用语法 `T&&` 创建，其中 `T` 是模板类型参数，或者使用 `auto&&`。这使得*完美转发*成为可能：在传递参数的同时保持它们的值类别（例如左值仍然是左值，临时对象被作为右值转发）。

转发引用允许引用根据类型绑定到左值或右值。转发引用遵循*引用折叠*规则：
* `T& &` 变成 `T&`
* `T& &&` 变成 `T&`
* `T&& &` 变成 `T&`
* `T&& &&` 变成 `T&&`

`auto` 在左值和右值上的类型推导：
```c++
int x = 0; // `x` is an lvalue of type `int`
auto&& al = x; // `al` is an lvalue of type `int&` -- binds to the lvalue, `x`
auto&& ar = 0; // `ar` is an lvalue of type `int&&` -- binds to the rvalue temporary, `0`
```

模板类型参数在左值和右值上的推导：
```c++
// Since C++14 or later:
void f(auto&& t) {
  // ...
}

// Since C++11 or later:
template <typename T>
void f(T&& t) {
  // ...
}

int x = 0;
f(0); // T is int, deduces as f(int &&) => f(int&&)
f(x); // T is int&, deduces as f(int& &&) => f(int&)

int& y = x;
f(y); // T is int&, deduces as f(int& &&) => f(int&)

int&& z = 0; // NOTE: `z` is an lvalue with type `int&&`.
f(z); // T is int&, deduces as f(int& &&) => f(int&)
f(std::move(z)); // T is int, deduces as f(int &&) => f(int&&)
```

参见：[`std::move`](#stdmove)、[`std::forward`](#stdforward)、[`右值引用`](#右值引用)。

### 可变参数模板
`...` 语法用于创建*参数包*或展开参数包。模板*参数包*是一种可以接受零个或多个模板实参（非类型、类型或模板）的模板参数。至少含有一个参数包的模板称为*可变参数模板*。
```c++
template <typename... T>
struct arity {
  constexpr static int value = sizeof...(T);
};
static_assert(arity<>::value == 0);
static_assert(arity<char, short, int>::value == 3);
```

它一个有趣的用法是从*参数包*创建*初始化器列表*，以便遍历可变参数的函数实参。
```c++
template <typename First, typename... Args>
auto sum(const First first, const Args... args) -> decltype(first) {
  const auto values = {first, args...};
  return std::accumulate(values.begin(), values.end(), First{0});
}

sum(1, 2, 3, 4, 5); // 15
sum(1, 2, 3);       // 6
sum(1.5, 2.0, 3.7); // 7.2
```

### 初始化器列表
一种使用“花括号列表”语法创建的轻量级、类数组的元素容器。例如，`{ 1, 2, 3 }` 创建一个整数序列，其类型为 `std::initializer_list<int>`。它适合用来替代向函数传递对象 vector 的做法。
```c++
int sum(const std::initializer_list<int>& list) {
  int total = 0;
  for (auto& e : list) {
    total += e;
  }

  return total;
}

auto list = {1, 2, 3};
sum(list); // == 6
sum({1, 2, 3}); // == 6
sum({}); // == 0
```

### 静态断言
在编译期求值的断言。
```c++
constexpr int x = 0;
constexpr int y = 1;
static_assert(x == y, "x != y");
```

### auto
`auto` 类型的变量由编译器根据其初始化器的类型推导得出。
```c++
auto a = 3.14; // double
auto b = 1; // int
auto& c = b; // int&
auto d = { 0 }; // std::initializer_list<int>
auto&& e = 1; // int&&
auto&& f = b; // int&
auto g = new auto(123); // int*
const auto h = 1; // const int
auto i = 1, j = 2, k = 3; // int, int, int
auto l = 1, m = true, n = 1.61; // error -- `l` deduced to be int, `m` is bool
auto o; // error -- `o` requires initializer
```

对于可读性极为有用，尤其是面对复杂类型时：
```c++
std::vector<int> v = ...;
std::vector<int>::const_iterator cit = v.cbegin();
// vs.
auto cit = v.cbegin();
```

函数也可以使用 `auto` 推导返回类型。在 C++11 中，返回类型必须显式指定，或者像这样使用 `decltype`：
```c++
template <typename X, typename Y>
auto add(X x, Y y) -> decltype(x + y) {
  return x + y;
}
add(1, 2); // == 3
add(1, 2.0); // == 3.0
add(1.5, 1.5); // == 3.0
```
上面例子中的尾置返回类型是表达式 `x + y` 的*声明类型*（参见 [`decltype`](#decltype) 一节）。例如，如果 `x` 是整数而 `y` 是 double，那么 `decltype(x + y)` 就是 double。因此，上面的函数会根据表达式 `x + y` 产生的类型来推导类型。注意，尾置返回类型可以访问其参数，在适当的时候也可以访问 `this`。

### lambda 表达式
lambda 是一个能够捕获作用域中变量的无名函数对象。它包含：一个*捕获列表*；一组可选的参数以及可选的尾置返回类型；以及一个函数体。捕获列表的示例：
* `[]` - 不捕获任何东西。
* `[=]` - 按值捕获作用域中的局部对象（局部变量、参数）。
* `[&]` - 按引用捕获作用域中的局部对象（局部变量、参数）。
* `[this]` - 按引用捕获 `this`。
* `[a, &b]` - 按值捕获对象 `a`，按引用捕获 `b`。

```c++
int x = 1;

auto getX = [=] { return x; };
getX(); // == 1

auto addX = [=](int y) { return x + y; };
addX(1); // == 2

auto getXRef = [&]() -> int& { return x; };
getXRef(); // int& to `x`
```
默认情况下，按值捕获的内容不能在 lambda 内部修改，因为编译器生成的方法被标记为 `const`。`mutable` 关键字允许修改被捕获的变量。该关键字放在参数列表之后（即使参数列表为空，也必须写出）。
```c++
int x = 1;

auto f1 = [&x] { x = 2; }; // OK: x is a reference and modifies the original

auto f2 = [x] { x = 2; }; // ERROR: the lambda can only perform const-operations on the captured value
// vs.
auto f3 = [x]() mutable { x = 2; }; // OK: the lambda can perform any operations on the captured value
```

### decltype
`decltype` 是一个运算符，它返回传给它的表达式的*声明类型*。如果 cv 限定符和引用是表达式的一部分，它们会被保留。`decltype` 的示例：
```c++
int a = 1; // `a` is declared as type `int`
decltype(a) b = a; // `decltype(a)` is `int`
const int& c = a; // `c` is declared as type `const int&`
decltype(c) d = a; // `decltype(c)` is `const int&`
decltype(123) e = 123; // `decltype(123)` is `int`
int&& f = 1; // `f` is declared as type `int&&`
decltype(f) g = 1; // `decltype(f) is `int&&`
decltype((a)) h = g; // `decltype((a))` is int&
```
```c++
template <typename X, typename Y>
auto add(X x, Y y) -> decltype(x + y) {
  return x + y;
}
add(1, 2.0); // `decltype(x + y)` => `decltype(3.0)` => `double`
```

参见：[`decltype(auto) (C++14)`](README.md#decltypeauto)。

### 类型别名
语义上与使用 `typedef` 类似，但使用 `using` 的类型别名更易读，并且与模板兼容。
```c++
template <typename T>
using Vec = std::vector<T>;
Vec<int> v; // std::vector<int>

using String = std::string;
String s {"foo"};
```

### nullptr
C++11 引入了一种新的空指针类型，旨在替代 C 的 `NULL` 宏。`nullptr` 本身的类型是 `std::nullptr_t`，可以隐式转换为指针类型，并且不同于 `NULL`，它不能转换为整型（除 `bool` 之外）。
```c++
void foo(int);
void foo(char*);
foo(NULL); // error -- ambiguous
foo(nullptr); // calls foo(char*)
```

### 强类型枚举
类型安全的枚举，解决了 C 风格枚举的诸多问题，包括：隐式转换、无法指定底层类型、作用域污染。
```c++
// Specifying underlying type as `unsigned int`
enum class Color : unsigned int { Red = 0xff0000, Green = 0xff00, Blue = 0xff };
// `Red`/`Green` in `Alert` don't conflict with `Color`
enum class Alert : bool { Red, Green };
Color c = Color::Red;
```

### 属性
属性为 `__attribute__(...)`、`__declspec` 等提供了一种通用的语法。
```c++
// `noreturn` attribute indicates `f` doesn't return.
[[ noreturn ]] void f() {
  throw "error";
}
```

### constexpr
常量表达式是指*可能*由编译器在编译期求值的表达式。常量表达式中只能进行非复杂的计算（这些规则在后来的版本中被逐步放宽）。使用 `constexpr` 说明符来表明变量、函数等是常量表达式。
```c++
constexpr int square(int x) {
  return x * x;
}

int square2(int x) {
  return x * x;
}

int a = square(2);  // mov DWORD PTR [rbp-4], 4

int b = square2(2); // mov edi, 2
                    // call square2(int)
                    // mov DWORD PTR [rbp-8], eax
```
在上面的代码片段中，注意调用 `square` 时的计算是在编译期完成的，然后结果被直接嵌入到生成的代码中，而 `square2` 则是在运行时被调用。

`constexpr` 值是编译器能够在编译期求值的值，但并不保证一定在编译期求值：
```c++
const int x = 123;
constexpr const int& y = x; // error -- constexpr variable `y` must be initialized by a constant expression
```

带类的常量表达式：
```c++
struct Complex {
  constexpr Complex(double r, double i) : re{r}, im{i} { }
  constexpr double real() { return re; }
  constexpr double imag() { return im; }

private:
  double re;
  double im;
};

constexpr Complex I(0, 1);
```

### 委托构造函数
构造函数现在可以使用初始化器列表来调用同一个类中的其他构造函数。
```c++
struct Foo {
  int foo;
  Foo(int foo) : foo{foo} {}
  Foo() : Foo(0) {}
};

Foo foo;
foo.foo; // == 0
```

### 用户定义字面量
用户定义字面量允许你扩展语言并添加自己的语法。要创建字面量，需要定义一个 `T operator "" X(...) { ... }` 函数，它返回类型 `T`，名字为 `X`。注意，这个函数的名字就定义了字面量的名字。任何不以 下划线开头的字面量名都是保留的，不会被调用。关于用户定义字面量函数应该接受什么参数，取决于字面量是在什么类型上调用的，这里有相应的规则。

将摄氏度转换为华氏度：
```c++
// `unsigned long long` parameter required for integer literal.
long long operator "" _celsius(unsigned long long tempCelsius) {
  return std::llround(tempCelsius * 1.8 + 32);
}
24_celsius; // == 75
```

字符串转整数：
```c++
// `const char*` and `std::size_t` required as parameters.
int operator "" _int(const char* str, std::size_t) {
  return std::stoi(str);
}

"123"_int; // == 123, with type `int`
```

### 显式虚函数覆盖
指定某个虚函数覆盖了另一个虚函数。如果该虚函数并没有覆盖父类的虚函数，就会产生编译错误。
```c++
struct A {
  virtual void foo();
  void bar();
};

struct B : A {
  void foo() override; // correct -- B::foo overrides A::foo
  void bar() override; // error -- A::bar is not virtual
  void baz() override; // error -- B::baz does not override A::baz
};
```

### final 说明符
指定某个虚函数不能在派生类中被覆盖，或者某个类不能被继承。
```c++
struct A {
  virtual void foo();
};

struct B : A {
  virtual void foo() final;
};

struct C : B {
  virtual void foo(); // error -- declaration of 'foo' overrides a 'final' function
};
```

类不能被继承。
```c++
struct A final {};
struct B : A {}; // error -- base 'A' is marked 'final'
```

### 默认函数
一种更优雅、更高效地为函数（例如构造函数）提供默认实现的方式。
```c++
struct A {
  A() = default;
  A(int x) : x{x} {}
  int x {1};
};
A a; // a.x == 1
A a2 {123}; // a.x == 123
```

配合继承：
```c++
struct B {
  B() : x{1} {}
  int x;
};

struct C : B {
  // Calls B::B
  C() = default;
};

C c; // c.x == 1
```

### 删除函数
一种更优雅、更高效地为函数提供“已删除”实现的方式。对于阻止对象被复制很有用。
```c++
class A {
  int x;

public:
  A(int x) : x{x} {};
  A(const A&) = delete;
  A& operator=(const A&) = delete;
};

A x {123};
A y = x; // error -- call to deleted copy constructor
y = x; // error -- operator= deleted
```

### 基于范围的 for 循环
用于遍历容器元素的语法糖。
```c++
std::array<int, 5> a {1, 2, 3, 4, 5};
for (int& x : a) x *= 2;
// a == { 2, 4, 6, 8, 10 }
```

注意使用 `int` 与使用 `int&` 时的区别：
```c++
std::array<int, 5> a {1, 2, 3, 4, 5};
for (int x : a) x *= 2;
// a == { 1, 2, 3, 4, 5 }
```

### 用于移动语义的特殊成员函数
进行拷贝时会调用拷贝构造函数和拷贝赋值运算符，而随着 C++11 引入移动语义，现在有了用于移动的移动构造函数和移动赋值运算符。
```c++
struct A {
  std::string s;
  A() : s{"test"} {}
  A(const A& o) : s{o.s} {}
  A(A&& o) : s{std::move(o.s)} {}
  A& operator=(A&& o) {
   s = std::move(o.s);
   return *this;
  }
};

A f(A a) {
  return a;
}

A a1 = f(A{}); // move-constructed from rvalue temporary
A a2 = std::move(a1); // move-constructed using std::move
A a3 = A{};
a2 = std::move(a3); // move-assignment using std::move
a1 = f(A{}); // move-assignment from rvalue temporary
```

### 转换构造函数
转换构造函数会把花括号列表语法中的值转换为构造函数的实参。
```c++
struct A {
  A(int) {}
  A(int, int) {}
  A(int, int, int) {}
};

A a {0, 0}; // calls A::A(int, int)
A b(0, 0); // calls A::A(int, int)
A c = {0, 0}; // calls A::A(int, int)
A d {0, 0, 0}; // calls A::A(int, int, int)
```

注意，花括号列表语法不允许窄化转换：
```c++
struct A {
  A(int) {}
};

A a(1.1); // OK
A b {1.1}; // Error narrowing conversion from double to int
```

注意，如果某个构造函数接受 `std::initializer_list`，那么它会被优先调用：
```c++
struct A {
  A(int) {}
  A(int, int) {}
  A(int, int, int) {}
  A(std::initializer_list<int>) {}
};

A a {0, 0}; // calls A::A(std::initializer_list<int>)
A b(0, 0); // calls A::A(int, int)
A c = {0, 0}; // calls A::A(std::initializer_list<int>)
A d {0, 0, 0}; // calls A::A(std::initializer_list<int>)
```

### 显式转换函数
转换函数现在可以使用 `explicit` 说明符变为显式的。
```c++
struct A {
  operator bool() const { return true; }
};

struct B {
  explicit operator bool() const { return true; }
};

A a;
if (a); // OK calls A::operator bool()
bool ba = a; // OK copy-initialization selects A::operator bool()

B b;
if (b); // OK calls B::operator bool()
bool bb = b; // error copy-initialization does not consider B::operator bool()
```
### 内联命名空间
内联命名空间的所有成员都被当作其父命名空间的成员一样对待，这允许对函数进行特化，并简化了版本管理的过程。这是一个传递性质：如果 A 包含 B，B 又包含 C，且 B 和 C 都是内联命名空间，那么 C 的成员就可以像在 A 中一样使用。

```c++
namespace Program {
  namespace Version1 {
    int getVersion() { return 1; }
    bool isFirstVersion() { return true; }
  }
  inline namespace Version2 {
    int getVersion() { return 2; }
  }
}

int version {Program::getVersion()};              // Uses getVersion() from Version2
int oldVersion {Program::Version1::getVersion()}; // Uses getVersion() from Version1
bool firstVersion {Program::isFirstVersion()};    // Does not compile when Version2 is added
```

### 非静态数据成员初始化器
允许非静态数据成员在声明处进行初始化，从而有可能清理掉构造函数中的默认初始化代码。

```c++
// Default initialization prior to C++11
class Human {
    Human() : age{0} {}
  private:
    unsigned age;
};
// Default initialization on C++11
class Human {
  private:
    unsigned age {0};
};
```

### 右尖括号
C++11 现在能够推断出一连串右尖括号是用作运算符还是用作 typedef 的结束符号，而无需添加空格。

```c++
typedef std::map<int, std::map <int, std::map <int, int> > > cpp98LongTypedef;
typedef std::map<int, std::map <int, std::map <int, int>>>   cpp11LongTypedef;
```

### 带引用限定的成员函数
成员函数现在可以根据 `*this` 是左值引用还是右值引用来进行限定。

```c++
struct Bar {
  // ...
};

struct Foo {
  Bar& getBar() & { return bar; }
  const Bar& getBar() const& { return bar; }
  Bar&& getBar() && { return std::move(bar); }
  const Bar&& getBar() const&& { return std::move(bar); }
private:
  Bar bar;
};

Foo foo{};
Bar bar = foo.getBar(); // calls `Bar& getBar() &`

const Foo foo2{};
Bar bar2 = foo2.getBar(); // calls `Bar& Foo::getBar() const&`

Foo{}.getBar(); // calls `Bar&& Foo::getBar() &&`
std::move(foo).getBar(); // calls `Bar&& Foo::getBar() &&`
std::move(foo2).getBar(); // calls `const Bar&& Foo::getBar() const&`
```

### 尾置返回类型
C++11 允许函数和 lambda 使用另一种语法来指定返回类型。
```c++
int f() {
  return 123;
}
// vs.
auto f() -> int {
  return 123;
}
```
```c++
auto g = []() -> int {
  return 123;
};
```
当某些返回类型无法被解析时，这个特性特别有用：
```c++
// NOTE: This does not compile!
template <typename T, typename U>
decltype(a + b) add(T a, U b) {
    return a + b;
}

// Trailing return types allows this:
template <typename T, typename U>
auto add(T a, U b) -> decltype(a + b) {
    return a + b;
}
```
在 C++14 中，可以改用 [`decltype(auto) (C++14)`](README.md#decltypeauto)。

### noexcept 说明符
`noexcept` 说明符指定函数是否可能抛出异常。它是 `throw()` 的改进版本。

```c++
void func1() noexcept;        // does not throw
void func2() noexcept(true);  // does not throw
void func3() throw();         // does not throw

void func4() noexcept(false); // may throw
```

不抛异常的函数允许调用可能抛异常的函数。每当有异常抛出，而处理程序的查找遇到不抛异常函数的最外层块时，就会调用函数 std::terminate。

```c++
extern void f();  // potentially-throwing
void g() noexcept {
    f();          // valid, even if f throws
    throw 42;     // valid, effectively a call to std::terminate
}
```

### char32_t 和 char16_t
为表示 UTF-8 字符串提供了标准类型。
```c++
char32_t utf8_str[] = U"\u0123";
char16_t utf8_str[] = u"\u0123";
```

### 原始字符串字面量
C++11 引入了一种将字符串字面量声明为“原始字符串字面量”的新方式。由转义序列产生的字符（制表符、换行符、单个反斜杠等）可以直接原样输入，同时保留格式。这在例如撰写可能包含大量引号或特殊格式的文学文本时很有用。这可以让你的字符串字面量更易于阅读和维护。

原始字符串字面量使用以下语法声明：
```
R"delimiter(raw_characters)delimiter"
```
其中：
* `delimiter` 是一个可选的字符序列，由除圆括号、反斜杠和空格之外的任意源字符组成。
* `raw_characters` 是任意原始字符序列；不能包含结束序列 `")delimiter"`。

示例：
```cpp
// msg1 and msg2 are equivalent.
const char* msg1 = "\nHello,\n\tworld!\n";
const char* msg2 = R"(
Hello,
	world!
)";
```

## C++11 库特性

### std::move
`std::move` 表示传给它的对象其资源可能被转移。使用被移动过的对象时应当小心，因为它们可能处于未指定的状态（参见：[What can I do with a moved-from object?](http://stackoverflow.com/questions/7027523/what-can-i-do-with-a-moved-from-object)）。

`std::move` 的一种定义（执行移动不过就是转换为右值引用）：
```c++
template <typename T>
typename remove_reference<T>::type&& move(T&& arg) {
  return static_cast<typename remove_reference<T>::type&&>(arg);
}
```

转移 `std::unique_ptr`：
```c++
std::unique_ptr<int> p1 {new int{0}};  // in practice, use std::make_unique
std::unique_ptr<int> p2 = p1; // error -- cannot copy unique pointers
std::unique_ptr<int> p3 = std::move(p1); // move `p1` into `p3`
                                         // now unsafe to dereference object held by `p1`
```

### std::forward
返回传给它的参数，同时保持参数的值类别和 cv 限定符。对于泛型代码和工厂函数很有用。它通常与[转发引用](#转发引用)配合使用。

`std::forward` 的一种定义：
```c++
template <typename T>
T&& forward(typename remove_reference<T>::type& arg) {
  return static_cast<T&&>(arg);
}
```

下面是一个函数 `wrapper` 的例子，它只是把其他的 `A` 对象转发给一个新的 `A` 对象的拷贝构造函数或移动构造函数：
```c++
struct A {
  A() = default;
  A(const A& o) { std::cout << "copied" << std::endl; }
  A(A&& o) { std::cout << "moved" << std::endl; }
};

template <typename T>
A wrapper(T&& arg) {
  return A{std::forward<T>(arg)};
}

wrapper(A{}); // moved
A a;
wrapper(a); // copied
wrapper(std::move(a)); // moved
```

参见：[`转发引用`](#转发引用)、[`右值引用`](#右值引用)。

### std::thread
`std::thread` 库提供了一种控制线程的标准方式，例如创建和终止线程。在下面的例子中，创建了多个线程来执行不同的计算，然后程序等待它们全部结束。

```c++
void foo(bool clause) { /* do something... */ }

std::vector<std::thread> threadsVector;
threadsVector.emplace_back([]() {
  // Lambda function that will be invoked
});
threadsVector.emplace_back(foo, true);  // thread will run foo(true)
for (auto& thread : threadsVector) {
  thread.join(); // Wait for threads to finish
}
```

### std::to_string
将数值实参转换为 `std::string`。
```c++
std::to_string(1.2); // == "1.2"
std::to_string(123); // == "123"
```

### 类型萃取
类型萃取（type traits）定义了一套编译期、基于模板的接口，用于查询或修改类型的属性。
```c++
static_assert(std::is_integral<int>::value);
static_assert(std::is_same<int, int>::value);
static_assert(std::is_same<std::conditional<true, int, double>::type, int>::value);
```

### 智能指针
C++11 引入了新的智能指针：`std::unique_ptr`、`std::shared_ptr`、`std::weak_ptr`。`std::auto_ptr` 现在已被弃用，并最终在 C++17 中被移除。

`std::unique_ptr` 是一个不可复制、可移动的指针，它管理自己堆分配的内存。**注意：优先使用 `std::make_X` 辅助函数，而不是直接使用构造函数。参见 [std::make_unique](https://github.com/AnthonyCalandra/modern-cpp-features/blob/master/CPP14.md#stdmake_unique) 和 [std::make_shared](#stdmake_shared) 两节。**
```c++
std::unique_ptr<Foo> p1 { new Foo{} };  // `p1` owns `Foo`
if (p1) {
  p1->bar();
}

{
  std::unique_ptr<Foo> p2 {std::move(p1)};  // Now `p2` owns `Foo`
  f(*p2);

  p1 = std::move(p2);  // Ownership returns to `p1` -- `p2` gets destroyed
}

if (p1) {
  p1->bar();
}
// `Foo` instance is destroyed when `p1` goes out of scope
```

`std::shared_ptr` 是一种智能指针，它管理被多个所有者共享的资源。共享指针持有一个*控制块*，其中包含若干组成部分，例如被管理的对象和一个引用计数器。所有对控制块的访问都是线程安全的，但是对被管理对象本身的操作*不是*线程安全的。
```c++
void foo(std::shared_ptr<T> t) {
  // Do something with `t`...
}

void bar(std::shared_ptr<T> t) {
  // Do something with `t`...
}

void baz(std::shared_ptr<T> t) {
  // Do something with `t`...
}

std::shared_ptr<T> p1 {new T{}};
// Perhaps these take place in another threads?
foo(p1);
bar(p1);
baz(p1);
```

### std::chrono
chrono 库包含一组用于处理*时长*（durations）、*时钟*（clocks）和*时间点*（time points）的工具函数和类型。这个库的一个用例是给代码做基准测试：
```c++
std::chrono::time_point<std::chrono::steady_clock> start, end;
start = std::chrono::steady_clock::now();
// Some computations...
end = std::chrono::steady_clock::now();

std::chrono::duration<double> elapsed_seconds = end - start;
double t = elapsed_seconds.count(); // t number of seconds, represented as a `double`
```

### 元组
元组是固定大小的异构值集合。可以通过使用 [`std::tie`](#stdtie) 解包，或者使用 `std::get` 来访问 `std::tuple` 的元素。
```c++
// `playerProfile` has type `std::tuple<int, const char*, const char*>`.
auto playerProfile = std::make_tuple(51, "Frans Nielsen", "NYI");
std::get<0>(playerProfile); // 51
std::get<1>(playerProfile); // "Frans Nielsen"
std::get<2>(playerProfile); // "NYI"
```

### std::tie
创建一个左值引用的元组。对于解包 `std::pair` 和 `std::tuple` 对象很有用。使用 `std::ignore` 作为被忽略值的占位符。在 C++17 中，应该改用结构化绑定。
```c++
// With tuples...
std::string playerName;
std::tie(std::ignore, playerName, std::ignore) = std::make_tuple(91, "John Tavares", "NYI");

// With pairs...
std::string yes, no;
std::tie(yes, no) = std::make_pair("yes", "no");
```

### std::array
`std::array` 是构建在 C 风格数组之上的容器。支持常见的容器操作，例如排序。
```c++
std::array<int, 3> a = {2, 1, 3};
std::sort(a.begin(), a.end()); // a == { 1, 2, 3 }
for (int& x : a) x *= 2; // a == { 2, 4, 6 }
```

### 无序容器
这些容器在查找、插入和删除操作上保持平均常数时间复杂度。为了达到常数时间复杂度，它们牺牲了顺序来换取速度，即通过哈希把元素分到不同的桶中。共有四种无序容器：
* `unordered_set`
* `unordered_multiset`
* `unordered_map`
* `unordered_multimap`

### std::make_shared
`std::make_shared` 是创建 `std::shared_ptr` 实例的推荐方式，原因如下：
* 避免使用 `new` 运算符。
* 在指定指针所持有的底层类型时，避免代码重复。
* 它提供了异常安全性。假设我们像下面这样调用函数 `foo`：
```c++
foo(std::shared_ptr<T>{new T{}}, function_that_throws(), std::shared_ptr<T>{new T{}});
```
编译器可以自由地先调用 `new T{}`，再调用 `function_that_throws()`，等等……由于我们在第一次构造 `T` 时已经在堆上分配了数据，这里就引入了内存泄漏。使用 `std::make_shared`，我们就获得了异常安全性：
```c++
foo(std::make_shared<T>(), function_that_throws(), std::make_shared<T>());
```
* 避免进行两次分配。当调用 `std::shared_ptr{ new T{} }` 时，我们需要为 `T` 分配内存，然后还要在共享指针内部为控制块分配内存。

有关 `std::unique_ptr` 和 `std::shared_ptr` 的更多信息，参见[智能指针](#智能指针)一节。

### std::ref
`std::ref(val)` 用于创建持有 val 引用的 `std::reference_wrapper` 类型对象。适用于使用 `&` 进行常规引用传递无法通过编译，或者 `&` 因类型推导而被丢弃的情况。`std::cref` 类似，但它创建的引用包装器持有对 val 的常量引用。

```c++
// create a container to store reference of objects.
auto val = 99;
auto _ref = std::ref(val);
_ref++;
auto _cref = std::cref(val);
//_cref++; does not compile
std::vector<std::reference_wrapper<int>>vec; // vector<int&>vec does not compile
vec.push_back(_ref); // vec.push_back(&i) does not compile
cout << val << endl; // prints 100
cout << vec[0] << endl; // prints 100
cout << _cref; // prints 100
```

### 内存模型
C++11 为 C++ 引入了内存模型，这意味着对线程和原子操作的库支持。其中一些操作包括（但不限于）原子加载/存储、比较并交换（compare-and-swap）、原子标志、promise、future、锁以及条件变量。

参见：[std::thread](#stdthread) 一节。

### std::async
`std::async` 以异步方式或惰性求值方式运行给定的函数，然后返回一个持有该函数调用结果的 `std::future`。

第一个参数是策略，可以是：
1. `std::launch::async | std::launch::deferred` 由实现决定是执行异步执行还是惰性求值。
1. `std::launch::async` 在新线程上运行可调用对象。
1. `std::launch::deferred` 在当前线程上执行惰性求值。

```c++
int foo() {
  /* Do something here, then return the result. */
  return 1000;
}

auto handle = std::async(std::launch::async, foo);  // create an async task
auto result = handle.get();  // wait for the result
```

### std::begin/end
新增了 `std::begin` 和 `std::end` 自由函数，用于以通用方式返回容器的起始和结束迭代器。这些函数也能用于没有 `begin` 和 `end` 成员函数的原始数组。

```c++
template <typename T>
int CountTwos(const T& container) {
  return std::count_if(std::begin(container), std::end(container), [](int item) {
    return item == 2;
  });
}

std::vector<int> vec = {2, 2, 43, 435, 4543, 534};
int arr[8] = {2, 43, 45, 435, 32, 32, 32, 32};
auto a = CountTwos(vec); // 2
auto b = CountTwos(arr);  // 1
```

## 致谢
* [cppreference](http://en.cppreference.com/w/cpp) - 对查找新库特性的示例和文档特别有用。
* [C++ Rvalue References Explained](http://web.archive.org/web/20240324121501/http://thbecker.net/articles/rvalue_references/section_01.html) - 我用来理解右值引用、完美转发和移动语义的优秀入门资料。
* [clang](http://clang.llvm.org/cxx_status.html) 和 [gcc](https://gcc.gnu.org/projects/cxx-status.html) 的标准支持页面。其中还包含了语言/库特性的提案，我借助这些提案来了解相关特性的描述、它要解决的问题以及一些示例。
* [Compiler explorer](https://godbolt.org/)
* [Scott Meyers 的《Effective Modern C++》](https://www.amazon.com/Effective-Modern-Specific-Ways-Improve/dp/1491903996) - 强烈推荐的书！
* [Jason Turner 的 C++ Weekly](https://www.youtube.com/channel/UCxHAlbZQNFU2LgEtiqd2Maw) - 优秀的 C++ 相关视频合集。
* [What can I do with a moved-from object?](http://stackoverflow.com/questions/7027523/what-can-i-do-with-a-moved-from-object)
* [What are some uses of decltype(auto)?](http://stackoverflow.com/questions/24109737/what-are-some-uses-of-decltypeauto)
* 以及许多我已经忘记的 Stack Overflow 帖子……

## 作者
Anthony Calandra

## 内容贡献者
参见：https://github.com/AnthonyCalandra/modern-cpp-features/graphs/contributors

## 许可证
MIT
