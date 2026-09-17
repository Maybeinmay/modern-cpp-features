# C++20

## 概述
本文中的许多描述和示例来自各种资源（参见 [致谢](#致谢) 部分），并由我用自己的话进行了总结。

C++20 包含以下新的语言特性：
- [协程](#协程)
- [概念](#概念concepts)
- [三路比较](#三路比较)
- [指定初始化器](#指定初始化器)
- [lambda 的模板语法](#lambda-的模板语法)
- [带初始化器的基于范围的 for 循环](#带初始化器的基于范围的-for-循环)
- [\[\[likely\]\] 和 \[\[unlikely\]\] 属性](#likely-和-unlikely-属性)
- [弃用对 this 的隐式捕获](#弃用对-this-的隐式捕获)
- [非类型模板参数中的类类型](#非类型模板参数中的类类型)
- [constexpr 虚函数](#constexpr-虚函数)
- [explicit(bool)](#explicitbool)
- [立即函数](#立即函数)
- [using enum](#using-enum)
- [lambda 捕获参数包](#lambda-捕获参数包)
- [char8_t](#char8_t)
- [constinit](#constinit)
- [`__VA_OPT__`](#__va_opt__)

C++20 包含以下新的库特性：
- [文本格式化](#文本格式化)
- [概念库](#概念库)
- [同步缓冲输出流](#同步缓冲输出流)
- [std::span](#stdspan)
- [位操作](#位操作)
- [数学常量](#数学常量)
- [std::is_constant_evaluated](#stdis_constant_evaluated)
- [std::make_shared 支持数组](#stdmake_shared-支持数组)
- [字符串的 starts_with 和 ends_with](#字符串的-starts_with-和-ends_with)
- [检查关联容器是否包含元素](#检查关联容器是否包含元素)
- [std::bit_cast](#stdbit_cast)
- [std::midpoint](#stdmidpoint)
- [std::to_array](#stdto_array)
- [std::bind_front](#stdbind_front)
- [统一的容器删除操作](#统一的容器删除操作)
- [三路比较辅助函数](#三路比较辅助函数)
- [std::lexicographical_compare_three_way](#stdlexicographical_compare_three_way)
- [std::jthread](#stdjthread)
- [安全的整型比较](#安全的整型比较)

## C++20 语言特性

### 协程

> **注意：** 虽然这些示例展示了如何以基本的方式使用协程，但代码编译时实际发生的事情要多得多。这些示例并不是对 C++20 协程的完整覆盖。由于标准库尚未提供 `generator` 和 `task` 类，我使用了 cppcoro 库来编译这些示例。

*协程*是可以被挂起和恢复的特殊函数。要定义协程，函数体中必须出现 `co_return`、`co_await` 或 `co_yield` 关键字。C++20 的协程是无栈的；除非被编译器优化掉，否则它们的状态分配在堆上。

协程的一个例子是*生成器*（generator）函数，它在每次调用时产生（即生成）一个值：
```c++
generator<int> range(int start, int end) {
  while (start < end) {
    co_yield start;
    start++;
  }

  // 在此函数末尾隐式执行 co_return：
  // co_return;
}

for (int n : range(0, 10)) {
  std::cout << n << std::endl;
}
```
上面的 `range` 生成器函数从 `start` 开始生成值，直到 `end`（不包含），每次迭代都产生当前存储在 `start` 中的值。生成器在 `range` 的每次调用之间保持其状态（在这里，调用对应着 for 循环的每一次迭代）。`co_yield` 接受给定的表达式，产出（即返回）其值，并在该点挂起协程。恢复时，执行会从 `co_yield` 之后继续。

协程的另一个例子是*任务*（task），它是一种异步计算，在被等待时执行：
```c++
task<void> echo(socket s) {
  for (;;) {
    auto data = co_await s.async_read();
    co_await async_write(s, data);
  }

  // 在此函数末尾隐式执行 co_return：
  // co_return;
}
```
在这个例子中引入了 `co_await` 关键字。该关键字接受一个表达式，如果你所等待的东西（这里是读或写）还没就绪，就挂起执行；否则就继续执行。（注意，在底层，`co_yield` 使用了 `co_await`。）

使用任务来惰性求值：
```c++
task<int> calculate_meaning_of_life() {
  co_return 42;
}

auto meaning_of_life = calculate_meaning_of_life();
// ...
co_await meaning_of_life; // == 42
```

### 概念（Concepts）
*概念*是具名的编译期谓词，用于约束类型。它们的形式如下：
```
template < template-parameter-list >
concept concept-name = constraint-expression;
```
其中 `constraint-expression` 求值为一个 constexpr 布尔值。*约束*应当表达语义上的要求，例如某个类型是否是数值类型或是否可哈希。如果给定类型不满足它所绑定的概念（即 `constraint-expression` 返回 `false`），就会产生编译错误。由于约束是在编译期求值的，它们可以提供更有意义的错误信息和运行时安全性。
```c++
// `T` 不受任何约束限制。
template <typename T>
concept always_satisfied = true;
// 把 `T` 限制为整型。
template <typename T>
concept integral = std::is_integral_v<T>;
// 把 `T` 同时限制为 `integral` 约束和有符号性。
template <typename T>
concept signed_integral = integral<T> && std::is_signed_v<T>;
// 把 `T` 同时限制为 `integral` 约束和 `signed_integral` 约束的否定。
template <typename T>
concept unsigned_integral = integral<T> && !signed_integral<T>;
```
有多种语法形式可以用来施加概念约束：
```c++
// 函数参数的形式：
// `T` 是一个受约束的类型模板参数。
template <my_concept T>
void f(T v);

// `T` 是一个受约束的类型模板参数。
template <typename T>
  requires my_concept<T>
void f(T v);

// `T` 是一个受约束的类型模板参数。
template <typename T>
void f(T v) requires my_concept<T>;

// `v` 是一个受约束的推导参数。
void f(my_concept auto v);

// `v` 是一个受约束的非类型模板参数。
template <my_concept auto v>
void g();

// auto 推导变量的形式：
// `foo` 是一个受约束的 auto 推导值。
my_concept auto foo = ...;

// lambda 的形式：
// `T` 是一个受约束的类型模板参数。
auto f = []<my_concept T> (T v) {
  // ...
};
// `T` 是一个受约束的类型模板参数。
auto f = []<typename T> requires my_concept<T> (T v) {
  // ...
};
// `T` 是一个受约束的类型模板参数。
auto f = []<typename T> (T v) requires my_concept<T> {
  // ...
};
// `v` 是一个受约束的推导参数。
auto f = [](my_concept auto v) {
  // ...
};
// `v` 是一个受约束的非类型模板参数。
auto g = []<my_concept auto v> () {
  // ...
};
```
`requires` 关键字用于开始一个 `requires` 子句或一个 `requires` 表达式：
```c++
template <typename T>
  requires my_concept<T> // `requires` 子句。
void f(T);

template <typename T>
concept callable = requires (T f) { f(); }; // `requires` 表达式。

template <typename T>
  requires requires (T x) { x + x; } // `requires` 子句和表达式在同一行。
T add(T a, T b) {
  return a + b;
}
```
注意，`requires` 表达式中的参数列表是可选的。`requires` 表达式中的每项要求都是以下几种之一：

* **简单要求** - 断言给定表达式是合法的。

```c++
template <typename T>
concept callable = requires (T f) { f(); };
```
* **类型要求** - 由 `typename` 关键字后跟一个类型名表示，断言给定的类型名是合法的。

```c++
struct foo {
  int foo;
};

struct bar {
  using value = int;
  value data;
};

struct baz {
  using value = int;
  value data;
};

// 使用 SFINAE，在 `T` 是 `baz` 时启用。
template <typename T, typename = std::enable_if_t<std::is_same_v<T, baz>>>
struct S {};

template <typename T>
using Ref = T&;

template <typename T>
concept C = requires {
                     // 对类型 `T` 的要求：
  typename T::value; // A) 有一个名为 `value` 的内部成员
  typename S<T>;     // B) 必须有一个对 `S` 合法的类模板特化
  typename Ref<T>;   // C) 必须是一次合法的别名模板替换
};

template <C T>
void g(T a);

g(foo{}); // 错误：不满足要求 A。
g(bar{}); // 错误：不满足要求 B。
g(baz{}); // 通过。
```
* **复合要求** - 花括号中的表达式，后跟一个尾置返回类型或类型约束。

```c++
template <typename T>
concept C = requires(T x) {
  {*x} -> std::convertible_to<typename T::inner>; // 表达式 `*x` 的类型可转换为 `T::inner`
  {x + 1} -> std::same_as<int>; // 表达式 `x + 1` 满足 `std::same_as<decltype((x + 1))>`
  {x * 1} -> std::convertible_to<T>; // 表达式 `x * 1` 的类型可转换为 `T`
};
```
* **嵌套要求** - 由 `requires` 关键字表示，用于指定附加约束（例如对局部参数实参的约束）。

```c++
template <typename T>
concept C = requires(T x) {
  requires std::same_as<sizeof(x), size_t>;
};
```
参见：[概念库](#概念库)。

### 三路比较
C++20 引入了三路比较运算符（`<=>`，即“宇宙飞船运算符”），作为一种编写比较函数的新方式，它可以减少样板代码，并帮助开发者定义更清晰的比较语义。定义三路比较运算符会自动生成其他的比较运算符函数（即 `==`、`!=`、`<` 等）。

引入了三种排序类型：
* `std::strong_ordering`：强序区分的是元素相等（完全相同且可互换）。提供 `less`、`greater`、`equivalent` 和 `equal` 排序。比较的例子：在列表中查找特定值、整数的值、区分大小写的字符串。
* `std::weak_ordering`：弱序区分的是元素等价（不完全相同，但就比较而言可以互换）。提供 `less`、`greater` 和 `equivalent` 排序。比较的例子：不区分大小写的字符串、排序、只比较某个类的部分而非全部可见成员。
* `std::partial_ordering`：偏序遵循与弱序相同的原则，但包含了无法排序的情况。提供 `less`、`greater`、`equivalent` 和 `unordered` 排序。比较的例子：浮点值（例如 `NaN`）。

默认的三路比较运算符逐成员进行比较：
```c++
struct foo {
  int a;
  bool b;
  char c;

  // 先比较 `a`，再比较 `b`，然后比较 `c` ...
  friend auto operator<=>(const foo&) const = default;
};

foo f1{0, false, 'a'}, f2{0, true, 'b'};
f1 < f2; // == true
f1 == f2; // == false
f1 >= f2; // == false
```

你也可以定义自己的比较：
```c++
struct foo {
  int x;
  bool b;
  char c;

  friend std::strong_ordering operator<=>(const foo& other) const {
      return x <=> other.x;
  }
};

foo f1{0, false, 'a'}, f2{0, true, 'b'};
f1 < f2; // == false
f1 == f2; // == true
f1 >= f2; // == true
```

### 指定初始化器
C 风格的指定初始化器语法。任何未在设计初始化列表中显式列出的成员字段都会被默认初始化。
```c++
struct A {
  int x;
  int y;
  int z = 123;
};

A a {.x = 1, .z = 2}; // a.x == 1, a.y == 0, a.z == 2
```

### lambda 的模板语法
在 lambda 表达式中使用熟悉的模板语法。
```c++
auto f = []<typename T>(std::vector<T> v) {
  // ...
};
```

### 带初始化器的基于范围的 for 循环
这个特性简化了常见的代码模式，有助于保持作用域紧凑，并为常见的生命周期问题提供了优雅的解决方案。
```c++
for (auto v = std::vector{1, 2, 3}; auto& e : v) {
  std::cout << e;
}
// 打印 "123"
```

### \[\[likely\]\] 和 \[\[unlikely\]\] 属性
向优化器提供提示，说明被标记的语句有很高的概率被执行。
```c++
switch (n) {
case 1:
  // ...
  break;

[[likely]] case 2:  // 认为 n == 2 比 n 的任何其他值
  // ...            // 都更可能
  break;
}
```

如果 likely/unlikely 属性之一出现在 if 语句的右括号之后，它表示该分支的（复合）语句（即函数体）很可能/不太可能被执行。
```c++
int random = get_random_number_between_x_and_y(0, 3);
if (random > 0) [[likely]] {
  // if 语句的主体
  // ...
}
```

它也可以应用于迭代语句的复合语句（函数体）。
```c++
while (unlikely_truthy_condition) [[unlikely]] {
  // while 语句的主体
  // ...
}
```

### 弃用对 this 的隐式捕获
在使用 `[=]` 的 lambda 捕获中隐式捕获 `this` 现在已被弃用；更推荐使用 `[=, this]` 或 `[=, *this]` 进行显式捕获。
```c++
struct int_value {
  int n = 0;
  auto getter_fn() {
    // 差：
    // return [=]() { return n; };

    // 好：
    return [=, *this]() { return n; };
  }
};
```

### 非类型模板参数中的类类型
现在可以在非类型模板参数中使用类。作为模板实参传入的对象类型为 `const T`（其中 `T` 是该对象的类型），并且具有静态存储期。
```c++
struct foo {
  foo() = default;
  constexpr foo(int) {}
};

template <foo f = {}>
auto get_foo() {
  return f;
}

get_foo(); // 使用隐式构造函数
get_foo<foo{123}>();
```

### constexpr 虚函数
虚函数现在可以是 `constexpr` 并在编译期求值。`constexpr` 虚函数可以覆盖非 `constexpr` 虚函数，反之亦然。
```c++
struct X1 {
  virtual int f() const = 0;
};

struct X2: public X1 {
  constexpr virtual int f() const { return 2; }
};

struct X3: public X2 {
  virtual int f() const { return 3; }
};

struct X4: public X3 {
  constexpr virtual int f() const { return 4; }
};

constexpr X4 x4;
x4.f(); // == 4
```

### explicit(bool)
在编译期有条件地选择构造函数是否为 explicit。`explicit(true)` 等同于直接指定 `explicit`。
```c++
struct foo {
  // 指定非整型类型（字符串、浮点数等）需要显式构造。
  template <typename T>
  explicit(!std::is_integral_v<T>) foo(T) {}
};

foo a = 123; // 正确
foo b = "123"; // 错误：explicit 构造函数不是候选（explicit 说明符求值为 true）
foo c {"123"}; // 正确
```

### 立即函数
与 `constexpr` 函数类似，但带有 `consteval` 说明符的函数必须产生一个常量。这类函数称为*立即函数*。
```c++
consteval int sqr(int n) {
  return n * n;
}

constexpr int r = sqr(100); // 正确
int x = 100;
int r2 = sqr(x); // 错误：'x' 的值不能用于常量表达式
                 // 如果 `sqr` 是 `constexpr` 函数则可以
```

### using enum
将枚举的成员引入作用域以提高可读性。之前：
```c++
enum class rgba_color_channel { red, green, blue, alpha };

std::string_view to_string(rgba_color_channel channel) {
  switch (channel) {
    case rgba_color_channel::red:   return "red";
    case rgba_color_channel::green: return "green";
    case rgba_color_channel::blue:  return "blue";
    case rgba_color_channel::alpha: return "alpha";
  }
}
```
之后：
```c++
enum class rgba_color_channel { red, green, blue, alpha };

std::string_view to_string(rgba_color_channel my_channel) {
  switch (my_channel) {
    using enum rgba_color_channel;
    case red:   return "red";
    case green: return "green";
    case blue:  return "blue";
    case alpha: return "alpha";
  }
}
```

### lambda 捕获参数包
按值捕获参数包：
```c++
template <typename... Args>
auto f(Args&&... args){
    // 按值：
    return [...args = std::forward<Args>(args)] {
        // ...
    };
}
```
按引用捕获参数包：
```c++
template <typename... Args>
auto f(Args&&... args){
    // 按引用：
    return [&...args = std::forward<Args>(args)] {
        // ...
    };
}
```

### char8_t
提供了表示 UTF-8 字符串的标准类型。
```c++
char8_t utf8_str[] = u8"\u0123";
```

### constinit
`constinit` 说明符要求变量必须在编译期初始化。
```c++
const char* g() { return "dynamic initialization"; }
constexpr const char* f() { return "constant initializer"; }

constinit const char* c = f();  // 正确
constinit const char* d = g();  // 错误：`g` 不是 constexpr，因此 `d` 无法在编译期求值。
```

### `__VA_OPT__`
通过在被展开时求值为给定实参（当可变参数宏非空时）来帮助支持可变参数宏。
```c++
#define F(...) f(0 __VA_OPT__(,) __VA_ARGS__)
F(a, b, c) // 被替换为 f(0, a, b, c)
F()        // 被替换为 f(0)
```

## C++20 库特性

### 文本格式化
通过 `std::format` 为标准库提供了编译期、有检查的字符串格式化能力。对于动态格式化字符串，也可以在运行时使用 `std::vformat` 及其他辅助工具进行文本格式化。文本格式化遵循给定的[规范](https://en.cppreference.com/w/cpp/utility/format/spec.html)。

`std::format` 接收一个格式字符串作为第一个参数，其后是数量可变的参数。如果格式化失败，编译就会失败：

```cpp
std::format("{}", 123); // 正确 -- 返回 "123"
std::format("{} {}", 123); // 错误 -- 参数不足
std::format("{} {}", "Here's a number:", 123); // 正确
```

基于运行时创建的格式器来格式化字符串：

```cpp
std::string fmt = "{} {}";
fmt += "{}{}";
std::vformat(fmt, std::make_format_args("Here's a number:", 1, 2, 3));
// 正确 -- 返回 "Here's a number: 123"
```

当格式化失败时（例如格式字符串非法），`std::vformat` 会抛出 `std::format_error`。

格式化自定义类型：

```c++
struct fraction {
  int numerator;
  int denominator;
};

template <>
struct std::formatter<fraction> {
  constexpr auto parse(std::format_parse_context& ctx) {
    return ctx.begin();
  }

  auto format(const fraction& f, std::format_context& ctx) const {
    return std::format_to(ctx.out(), "{0:d}/{1:d}", f.numerator, f.denominator);
  }
};

fraction f{1, 2};
std::format("{}", f); // == "1/2"
```

### 概念库
标准库也提供了一些概念，用于构建更复杂的概念。其中包括：

**核心语言概念：**
- `same_as` - 指定两个类型相同。
- `derived_from` - 指定一个类型派生自另一个类型。
- `convertible_to` - 指定一个类型可以隐式转换为另一个类型。
- `common_with` - 指定两个类型共享一个公共类型。
- `integral` - 指定一个类型是整型。
- `default_constructible` - 指定某个类型的对象可以被默认构造。

 **比较概念：**
- `boolean` - 指定一个类型可用于布尔语境。
- `equality_comparable` - 指定 `operator==` 是一个等价关系。

 **对象概念：**
- `movable` - 指定某个类型的对象可以被移动和交换。
- `copyable` - 指定某个类型的对象可以被复制、移动和交换。
- `semiregular` - 指定某个类型的对象可以被复制、移动、交换和默认构造。
- `regular` - 指定一个类型是*正则的*（regular），即它既是 `semiregular` 又是 `equality_comparable`。

 **可调用概念：**
- `invocable` - 指定某个可调用类型可以用给定的一组实参类型来调用。
- `predicate` - 指定某个可调用类型是布尔谓词。

参见：[概念](#概念concepts)。

### 同步缓冲输出流
为被包装的输出流缓冲输出操作，确保同步（即输出不会交错）。
```c++
std::osyncstream{std::cout} << "The value of x is:" << x << std::endl;
```

### std::span
span 是容器的一个视图（即非拥有视图），为一组连续元素提供带边界检查的访问。由于视图不拥有其元素，它们构造和复制都很廉价——理解视图的一种简化方式是：它们持有对其数据的引用。与分别维护指针/迭代器和长度字段不同，span 把两者封装在单个对象中。

span 可以是动态大小的，也可以是固定大小的（固定大小称为其 *extent*）。固定大小的 span 可以从边界检查中获益。

span 不会传播 const，因此要构造只读 span，请使用 `std::span<const T>`。

示例：使用动态大小的 span 打印来自各种容器的整数。
```c++
void print_ints(std::span<const int> ints) {
    for (const auto n : ints) {
        std::cout << n << std::endl;
    }
}

print_ints(std::vector{ 1, 2, 3 });
print_ints(std::array<int, 5>{ 1, 2, 3, 4, 5 });

int a[10] = { 0 };
print_ints(a);
// 等等
```

示例：静态大小的 span 对于 extent 不匹配的容器会编译失败。
```c++
void print_three_ints(std::span<const int, 3> ints) {
    for (const auto n : ints) {
        std::cout << n << std::endl;
    }
}

print_three_ints(std::vector{ 1, 2, 3 }); // 错误
print_three_ints(std::array<int, 5>{ 1, 2, 3, 4, 5 }); // 错误
int a[10] = { 0 };
print_three_ints(a); // 错误

std::array<int, 3> b = { 1, 2, 3 };
print_three_ints(b); // 正确

// 如果需要，你也可以手动构造 span：
std::vector c{ 1, 2, 3 };
print_three_ints(std::span<const int, 3>{ c.data(), 3 }); // 正确：设置指针和长度字段。
print_three_ints(std::span<const int, 3>{ c.cbegin(), c.cend() }); // 正确：使用迭代器对。
```

### 位操作
C++20 提供了新的 `<bit>` 头文件，其中包含一些位操作，包括 popcount。
```c++
std::popcount(0u); // 0
std::popcount(1u); // 1
std::popcount(0b1111'0000u); // 4
```

### 数学常量
在 `<numbers>` 头文件中定义的数学常量，包括 PI、欧拉数等。
```c++
std::numbers::pi; // 3.14159...
std::numbers::e; // 2.71828...
```

### std::is_constant_evaluated
一个谓词函数，在编译期语境中被调用时返回真值。
```c++
constexpr bool is_compile_time() {
    return std::is_constant_evaluated();
}

constexpr bool a = is_compile_time(); // true
bool b = is_compile_time(); // false
```

### std::make_shared 支持数组
```c++
auto p = std::make_shared<int[]>(5); // 指向 `int[5]` 的指针
// 或者
auto p = std::make_shared<int[5]>(); // 指向 `int[5]` 的指针
```

### 字符串的 starts_with 和 ends_with
字符串（以及 string view）现在有了 `starts_with` 和 `ends_with` 成员函数，用于检查字符串是否以给定的字符串开头或结尾。
```c++
std::string str = "foobar";
str.starts_with("foo"); // true
str.ends_with("baz"); // false
```

### 检查关联容器是否包含元素
像 set 和 map 这样的关联容器有了 `contains` 成员函数，可以用它来代替“查找并检查迭代器是否为 end”的惯用法。
```c++
std::map<int, char> map {{1, 'a'}, {2, 'b'}};
map.contains(2); // true
map.contains(123); // false

std::set<int> set {1, 2, 3};
set.contains(2); // true
```

### std::bit_cast
一种更安全的将一个对象从一种类型重新解释为另一种类型的方式。
```c++
float f = 123.0;
int i = std::bit_cast<int>(f);
```

### std::midpoint
安全地（不产生溢出地）计算两个整数的中点。
```c++
std::midpoint(1, 3); // == 2
```

### std::to_array
将给定的数组/“类数组”对象转换为 `std::array`。
```c++
std::to_array("foo"); // 返回 `std::array<char, 4>`
std::to_array<int>({1, 2, 3}); // 返回 `std::array<int, 3>`

int a[] = {1, 2, 3};
std::to_array(a); // 返回 `std::array<int, 3>`
```

### std::bind_front
将前 N 个参数（N 是传给 `std::bind_front` 的给定函数之后的参数个数）绑定到给定的自由函数、lambda 或成员函数上。
```c++
const auto f = [](int a, int b, int c) { return a + b + c; };
const auto g = std::bind_front(f, 1, 1);
g(1); // == 3
```

### 统一的容器删除操作
为各种 STL 容器（如 string、list、vector、map 等）提供 `std::erase` 和/或 `std::erase_if`。

按值删除使用 `std::erase`，而要指定何时删除元素的谓词则使用 `std::erase_if`。两个函数都返回被删除元素的数量。

```c++
std::vector v{0, 1, 0, 2, 0, 3};
std::erase(v, 0); // v == {1, 2, 3}
std::erase_if(v, [](int n) { return n == 0; }); // v == {1, 2, 3}
```

### 三路比较辅助函数
用于给比较结果命名的辅助函数：
```c++
std::is_eq(0 <=> 0); // == true
std::is_lteq(0 <=> 1); // == true
std::is_gt(0 <=> 1); // == false
```

参见：[三路比较](#三路比较)。

### std::lexicographical_compare_three_way
使用三路比较按字典序比较两个范围，并产生最强的、适用的比较类别类型的结果。
```c++
std::vector a{0, 0, 0}, b{0, 0, 0}, c{1, 1, 1};

auto cmp_ab = std::lexicographical_compare_three_way(
    a.begin(), a.end(), b.begin(), b.end());
std::is_eq(cmp_ab); // == true

auto cmp_ac = std::lexicographical_compare_three_way(
    a.begin(), a.end(), c.begin(), c.end());
std::is_lt(cmp_ac); // == true
```

参见：[三路比较](#三路比较)、[三路比较辅助函数](#三路比较辅助函数)。

### std::jthread
一种执行线程（类似 `std::thread`），它在析构时会进行 join，并且可以被通知停止。

与 `std::thread` 需要先检查线程是否可 join 然后再 join 不同，`std::jthread` 会在其析构函数中自动尝试 join。

与 `std::thread` 不同，你可以通过调用 `std::jthread::request_stop` 或通过线程的 `stop_source` 来请求它停止：

```cpp
std::jthread t{
    [](std::stop_token stoken) {
        while (!stoken.stop_requested()) {
            std::this_thread::sleep_for(1s);
        }
    }
};

// 从线程对象请求停止：
t.request_stop();
// 或者，通过停止源（stop source）：
std::stop_source stopSource = t.get_stop_source();
stopSource.request_stop();
```

`std::stop_token` 可以用来查询线程的停止状态。

### 安全的整型比较
比较整数（包括类型不同的整数），而不受整型转换的危险影响。

```cpp
-1 > 0U; // == true
std::cmp_greater(-1, 0U); // == false

std::cmp_equal(0U, 0); // == true
std::cmp_less_equal(-1, 1U); // == true

std::in_range<unsigned>(-1); // == false
std::in_range<char>(999999); // == false
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
