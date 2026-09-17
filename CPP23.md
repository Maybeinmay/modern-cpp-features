# C++23

## 概述
本文中的许多描述和示例来自各种资源（参见 [致谢](#致谢) 部分），并由我用自己的话进行了总结。

C++23 包含以下新的语言特性：
- [consteval if](#consteval-if)
- [推导 `this`](#推导-this)
- [多维下标运算符](#多维下标运算符)
- [提高基于范围的 `for` 循环的安全性](#提高基于范围的-for-循环的安全性)

C++23 包含以下新的库特性：
- [stacktrace 库](#stacktrace-库)
- [字符串和 string_view 的 `contains`](#字符串和-string_view-的-contains)
- [std::to_underlying](#stdto_underlying)
- [`spanstream`](#spanstream)
- [输入/输出指针](#输入输出指针)
- [`std::optional` 的单子操作](#stdoptional-的单子操作)
- [`std::expected`](#stdexpected)
- [`std::unreachable`](#stdunreachable)

## C++23 语言特性

### consteval if
编写在常量求值期间被实例化的代码。
```c++
consteval int f(int i) { return i; }

constexpr int g(int i) {
  if consteval {
      return f(i);
  } else {
      return 42;
  }
}
```

### 推导 `this`
借助 C++23 引入的显式对象成员函数，现在可以通过将成员函数的第一个参数以 `this` 关键字为前缀来推导对象的类型和值类别：
```c++
// NEW WAY USING DEDUCING THIS:
struct T {
  decltype(auto) operator[](this auto& self, std::size_t idx) { 
    return self.mVector[idx]; 
  }
};

// OLD WAY:
struct T {
  value_t& operator[](std::size_t idx) {
    return mVector[idx];
  }
  const value_t& operator[](std::size_t idx) const {
    return mVector[idx];
  }
};
```

### 多维下标运算符
为 `operator[]` 运算符指定零个或多个参数：
```c++
template <typename T, std::size_t Z, std::size_t Y, std::size_t X>
struct Array3d {
  std::array<T, X * Y * Z> m{};

  T& operator[](std::size_t z, std::size_t y, std::size_t x) {
      return m[z * Y * X + y * X + x];
  }
};

Array3d<int, 4, 3, 2> v;
v[3, 2, 1] = 42;
```

### 提高基于范围的 `for` 循环的安全性
修复了 C++ 中最重要的控制结构之一的一些臭名昭著的生命周期问题。

一些在 C++23 之前有问题、现在已被修复的代码片段示例：

* `for (auto e : getTmp().getRef())`
* `for (auto e : getVector()[0])`
* `for (auto valueElem : getMap()["key"])`
* `for (auto e : get<0>(getTuple()))`
* `for (auto e : getOptionalCollection().value())`
* `for (char c : get<std::string>(getVariant()))`

## C++23 库特性

### Stacktrace 库
栈回溯（stacktrace）是对调用序列的一种近似表示，由栈回溯条目组成。一个栈回溯条目（由 `std::stacktrace_entry` 表示）包含的信息包括源文件和行号，以及一个描述字段。

在 Linux 系统上的示例输出：
```c++
#include <print>
#include <stacktrace>

int main() {
    std::println("{}", std::stacktrace::current());
}
```
```
  0#  main at /app/example.cpp:5 [0x5ee42e3db747]
  1#  <unknown> [0x76e76dc29d8f]
  2#  __libc_start_main [0x76e76dc29e3f]
  3#  _start [0x5ee42e3db644]
```

### 字符串和 string_view 的 `contains`
一个更简单的函数，用于查询某个子串是否包含在字符串或 string_view 中：
```c++
std::string{"foobarbaz"}.contains("bar"); // == true
std::string{"foobarbaz"}.contains("bat"); // == false
```

### `std::to_underlying`
支持将枚举转换为其底层类型的常见工具：
```c++
enum class MyEnum : int { A = 1, B, C };
std::to_underlying(MyEnum::A); // == 1
std::to_underlying(MyEnum::C); // == 3
```

### `spanstream`
一个 `strstream` 的替代品，使用字符 span 作为外部提供的缓冲区。对缓冲区没有所有权，也不会重新分配。
```c++
char input[] = "10 20 30";
std::ispanstream is{std::span<char>{input}};
int i;
is >> i; // i == 10
is >> i; // i == 20
is >> i; // i == 30
```
```c++
char output[30]{}; // zero-initialize array
std::ospanstream os{std::span<char>{output}};
os << 10 << 20 << 30;
std::span<char> sp = os.span();
```

### 输入/输出指针
`std::out_ptr` 和 `std::inout_ptr` 是用于同时支持 C API 和智能指针的抽象：它们创建一个临时的二级指针，并在其析构时更新智能指针。简而言之：它是一个可转换为 `T**` 的东西，在离开作用域时会更新（通过 `reset` 调用或语义上等价的行为）它所基于的智能指针。

当抛出异常时，这个抽象还能安全地管理相关内存的生命周期。
```c++
// p_handle is written (out) to.
int c_api_create_handle(MyHandle** p_handle);
// p_handle is both read (in) and written (out) to.
int c_api_recreate_handle(MyHandle** p_handle);
void c_api_delete_handle(MyHandle* handle);

struct resource_deleter {
	void operator()(MyHandle* handle) {
		c_api_delete_handle(handle);
	}
};
```
```c++
std::unique_ptr<MyHandle, resource_deleter> resource(nullptr);
int err = c_api_create_handle(std::out_ptr(resource));
// `resource` now owns the memory allocated within `c_api_create_handle`.
```
```c++
std::shared_ptr<MyHandle> resource(nullptr);
int err = c_api_recreate_handle(std::inout_ptr(resource), resource_deleter{});
// `resource` now shares the memory allocated within `c_api_recreate_handle`.
```

inout/out 指针都支持（隐式地）转换为 `void**`，并支持显式地转换为用户指定的类型。

### `std::optional` 的单子操作
为 `std::optional` 支持各种 `and_then`、`transform` 和 `or_else` 操作。
```c++
std::optional<int> parse_int(const std::string&);
std::optional<int> ensure_non_negative(int);
std::optional<double> default_value_or_empty(double);

std::optional<double> stringToSqrtDouble(const std::string& input) {
  return parse_int(input)
    .and_then(ensure_non_negative)
    .transform([](int x) {
      return std::sqrt(x);
    })
    .or_else(default_value_or_empty);
}
```

### `std::expected`
`std::expected` 提供了一种方式，将值和潜在的错误值都包含在同一个类型中。它还支持对期望值和意外（即错误）值的一系列单子操作。

使用 `std::unexpected` 来存储一个意外（即错误）值。
```c++
enum class StringToSqrtDoubleError {
    ParseError, NegativeNumber
};

std::expected<int, StringToSqrtDoubleError> parse_int(const std::string&);

std::expected<double, StringToSqrtDoubleError> stringToSqrtDouble(const std::string& input) {
    auto parsed = parse_int(input);
    if (!parsed) return parsed;

    auto parsedInt = *parsed;
    if (parsedInt < 0) return std::unexpected(StringToSqrtDoubleError::NegativeNumber);

    return std::sqrt(parsedInt);
}
```

### `std::unreachable`
提供了一种显式地将某条代码路径标记为不可达的方式。如果该代码路径真的被执行到，可能会表现出未定义行为。
```c++
enum class MyEnum { A, B, C };

int convertMyEnumToInt(MyEnum e) {
    switch (e) {
        case MyEnum::A: return 0;
        case MyEnum::B: return 1;
        case MyEnum::C: return 2;
        default: std::unreachable(); 
    }
}
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
