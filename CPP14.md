# C++14

## 概述
本文中的许多描述和示例来自各种资源（参见 [致谢](#致谢) 部分），并由我用自己的话进行了总结。

C++14 包含以下新的语言特性：
- [二进制字面量](#二进制字面量)
- [泛型 lambda 表达式](#泛型-lambda-表达式)
- [lambda 捕获初始化器](#lambda-捕获初始化器)
- [返回类型推导](#返回类型推导)
- [decltype(auto)](#decltypeauto)
- [放宽对 constexpr 函数的约束](#放宽对-constexpr-函数的约束)
- [变量模板](#变量模板)
- [\[\[deprecated\]\] 属性](#deprecated-属性)

C++14 包含以下新的库特性：
- [标准库类型的用户定义字面量](#标准库类型的用户定义字面量)
- [编译期整数序列](#编译期整数序列)
- [std::make_unique](#stdmake_unique)

## C++14 语言特性

### 二进制字面量
二进制字面量提供了一种方便的方式来表示二进制数。
可以用 `'` 分隔数字。
```c++
0b110 // == 6
0b1111'1111 // == 255
```

### 泛型 lambda 表达式
C++14 现在允许在参数列表中使用 `auto` 类型说明符，从而支持多态 lambda。
```c++
auto identity = [](auto x) { return x; };
int three = identity(3); // == 3
std::string foo = identity("foo"); // == "foo"
```

### lambda 捕获初始化器
它允许创建用任意表达式初始化的 lambda 捕获。给被捕获值起的名字不必与外部作用域中的任何变量相关，它会在 lambda 函数体内部引入一个新名字。初始化表达式在 lambda 被*创建*时求值（而不是在其被*调用*时）。
```c++
int factory(int i) { return i * 10; }
auto f = [x = factory(2)] { return x; }; // 返回 20

auto generator = [x = 0] () mutable {
  // 如果不加 'mutable' 就无法编译，因为每次调用都会修改 x
  return x++;
};
auto a = generator(); // == 0
auto b = generator(); // == 1
auto c = generator(); // == 2
```
由于现在可以将值*移动*（或*转发*）到 lambda 中（而以前只能按复制或引用捕获），我们现在可以按值捕获仅可移动（move-only）的类型。注意，在下面的例子中，`task2` 的捕获列表中 `=` 左侧的 `p` 是一个仅属于 lambda 函数体的新变量，并不引用原来的 `p`。
```c++
auto p = std::make_unique<int>(1);

auto task1 = [=] { *p = 5; }; // 错误：std::unique_ptr 无法被复制
// 对比
auto task2 = [p = std::move(p)] { *p = 5; }; // OK：p 被移动构造进闭包对象
// task2 创建后，原来的 p 为空
```
使用这种方式，引用捕获可以与所引用的变量具有不同的名字。
```c++
auto x = 1;
auto f = [&r = x, x = x * 10] {
  ++r;
  return r + x;
};
f(); // 将 x 设为 2 并返回 12
```

### 返回类型推导
在 C++14 中使用 `auto` 返回类型，编译器会尝试为你推导类型。对于 lambda，现在可以使用 `auto` 推导其返回类型，这使得返回推导出的引用或右值引用成为可能。
```c++
// 推导返回类型为 `int`。
auto f(int i) {
 return i;
}
```
```c++
template <typename T>
auto& f(T& t) {
  return t;
}

// 返回所推导类型的引用。
auto g = [](auto& x) -> auto& { return f(x); };
int y = 123;
int& z = g(y); // `y` 的引用
```

### decltype(auto)
`decltype(auto)` 类型说明符也会像 `auto` 一样推导类型。但是，它在推导返回类型时会保留引用和 cv 限定符，而 `auto` 则不会。
```c++
const int x = 0;
auto x1 = x; // int
decltype(auto) x2 = x; // const int
int y = 0;
int& y1 = y;
auto y2 = y1; // int
decltype(auto) y3 = y1; // int&
int&& z = 0;
auto z1 = std::move(z); // int
decltype(auto) z2 = std::move(z); // int&&
```
```c++
// 注意：对泛型代码尤其有用！

// 返回类型是 `int`。
auto f(const int& i) {
 return i;
}

// 返回类型是 `const int&`。
decltype(auto) g(const int& i) {
 return i;
}

int x = 123;
static_assert(std::is_same<const int&, decltype(f(x))>::value == 0);
static_assert(std::is_same<int, decltype(f(x))>::value == 1);
static_assert(std::is_same<const int&, decltype(g(x))>::value == 1);
```

参见：[`decltype (C++11)`](README.md#decltype)。

### 放宽对 constexpr 函数的约束
在 C++11 中，`constexpr` 函数体只能包含非常有限的语法，包括（但不限于）`typedef`、`using` 以及单条 `return` 语句。在 C++14 中，允许的语法范围大大扩展，包括了最常见的语法，如 `if` 语句、多个 `return`、循环等。
```c++
constexpr int factorial(int n) {
  if (n <= 1) {
    return 1;
  } else {
    return n * factorial(n - 1);
  }
}
factorial(5); // == 120
```

### 变量模板
C++14 允许变量模板化：

```c++
template<class T>
constexpr T pi = T(3.1415926535897932385);
template<class T>
constexpr T e  = T(2.7182818284590452353);
```

### [[deprecated]] 属性
C++14 引入了 `[[deprecated]]` 属性，用于指示某个单元（函数、类等）已不推荐使用，并很可能产生编译警告。如果提供了原因，它会被包含在警告中。
```c++
[[deprecated]]
void old_method();
[[deprecated("Use new_method instead")]]
void legacy_method();
```

## C++14 库特性

### 标准库类型的用户定义字面量
为标准库类型新增了用户定义字面量，包括为 `chrono` 和 `basic_string` 新增的内置字面量。它们可以是 `constexpr`，即可以在编译期使用。这些字面量的一些用途包括编译期整数解析、二进制字面量以及虚数字面量。
```c++
using namespace std::chrono_literals;
auto day = 24h;
day.count(); // == 24
std::chrono::duration_cast<std::chrono::minutes>(day).count(); // == 1440
```

### 编译期整数序列
类模板 `std::integer_sequence` 表示一个编译期的整数序列。在其之上构建了几个辅助工具：
* `std::make_integer_sequence<T, N>` - 创建一个类型为 `T` 的 `0, ..., N - 1` 序列。
* `std::index_sequence_for<T...>` - 将一个模板参数包转换为整数序列。

将数组转换为元组：
```c++
template<typename Array, std::size_t... I>
decltype(auto) a2t_impl(const Array& a, std::integer_sequence<std::size_t, I...>) {
  return std::make_tuple(a[I]...);
}

template<typename T, std::size_t N, typename Indices = std::make_index_sequence<N>>
decltype(auto) a2t(const std::array<T, N>& a) {
  return a2t_impl(a, Indices());
}
```

### std::make_unique
`std::make_unique` 是创建 `std::unique_ptr` 实例的推荐方式，原因如下：
* 避免使用 `new` 运算符。
* 在指定指针所持有的底层类型时，避免代码重复。
* 最重要的是，它提供了异常安全性。假设我们像下面这样调用函数 `foo`：
```c++
foo(std::unique_ptr<T>{new T{}}, function_that_throws(), std::unique_ptr<T>{new T{}});
```
编译器可以自由地先调用 `new T{}`，再调用 `function_that_throws()`，等等……由于我们在第一次构造 `T` 时已经在堆上分配了数据，这里就引入了内存泄漏。使用 `std::make_unique`，我们就获得了异常安全性：
```c++
foo(std::make_unique<T>(), function_that_throws(), std::make_unique<T>());
```

有关 `std::unique_ptr` 和 `std::shared_ptr` 的更多信息，参见 [智能指针（C++11）](README.md#smart-pointers) 部分。

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
