# C++17

## 概述
本文中的许多描述和示例来自各种资源（参见 [致谢](#致谢) 部分），并由我用自己的话进行了总结。

C++17 包含以下新的语言特性：
- [类模板的模板实参推导](#类模板的模板实参推导)
- [用 auto 声明非类型模板参数](#用-auto-声明非类型模板参数)
- [折叠表达式](#折叠表达式)
- [从花括号初始化列表进行 auto 推导的新规则](#从花括号初始化列表进行-auto-推导的新规则)
- [constexpr lambda](#constexpr-lambda)
- [按值捕获 this 的 lambda](#按值捕获-this-的-lambda)
- [内联变量](#内联变量)
- [嵌套命名空间](#嵌套命名空间)
- [结构化绑定](#结构化绑定)
- [带初始化器的选择语句](#带初始化器的选择语句)
- [constexpr if](#constexpr-if)
- [UTF-8 字符字面量](#utf-8-字符字面量)
- [枚举的直接列表初始化](#枚举的直接列表初始化)
- [\[\[fallthrough\]\]、\[\[nodiscard\]\]、\[\[maybe_unused\]\] 属性](#fallthroughnodiscardmaybe_unused-属性)
- [`__has_include`](#__has_include)
- [类模板实参推导（CTAD）](#类模板实参推导ctad)

C++17 包含以下新的库特性：
- [std::variant](#stdvariant)
- [std::optional](#stdoptional)
- [std::any](#stdany)
- [std::string_view](#stdstring_view)
- [std::invoke](#stdinvoke)
- [std::apply](#stdapply)
- [std::filesystem](#stdfilesystem)
- [std::byte](#stdbyte)
- [map 和 set 的拼接（splicing）](#map-和-set-的拼接splicing)
- [并行算法](#并行算法)
- [std::sample](#stdsample)
- [std::clamp](#stdclamp)
- [std::reduce](#stdreduce)
- [前缀和算法](#前缀和算法)
- [GCD 和 LCM](#gcd-和-lcm)
- [std::not_fn](#stdnot_fn)
- [字符串与数字之间的转换](#字符串与数字之间的转换)
- [用于 chrono duration 和 timepoint 的取整函数](#用于-chrono-duration-和-timepoint-的取整函数)

## C++17 语言特性

### 类模板的模板实参推导
类似于函数模板的自动模板实参推导，但现在也包括了类的构造函数。
```c++
template <typename T = float>
struct MyContainer {
  T val;
  MyContainer() : val{} {}
  MyContainer(T val) : val{val} {}
  // ...
};
MyContainer c1 {1}; // OK：MyContainer<int>
MyContainer c2; // OK：MyContainer<float>
```

### 用 auto 声明非类型模板参数
遵循 `auto` 的推导规则，同时遵守允许作为非类型模板参数的类型列表[\*]，模板实参可以从其参数的类型中推导出来：
```c++
template <auto... seq>
struct my_integer_sequence {
  // 实现放在这里 ...
};

// 显式将类型 `int` 作为模板实参传入。
auto seq = std::integer_sequence<int, 0, 1, 2>();
// 类型被推导为 `int`。
auto seq2 = my_integer_sequence<0, 1, 2>();
```
\* - 例如，你不能把 `double` 用作模板参数类型，这也使得使用 `auto` 的这种推导无效。

### 折叠表达式
折叠表达式会对一个模板参数包按某个二元运算符进行折叠。
* 形如 `(... op e)` 或 `(e op ...)` 的表达式，其中 `op` 是折叠运算符，`e` 是未展开的参数包，称为*一元折叠*。
* 形如 `(e1 op ... op e2)` 的表达式，其中 `op` 是折叠运算符，称为*二元折叠*。`e1` 或 `e2` 中有一个是未展开的参数包，但不能两个都是。
```c++
template <typename... Args>
bool logicalAnd(Args... args) {
    // 二元折叠。
    return (true && ... && args);
}
bool b = true;
bool& b2 = b;
logicalAnd(b, b2, true); // == true
```
```c++
template <typename... Args>
auto sum(Args... args) {
    // 一元折叠。
    return (... + args);
}
sum(1.0, 2.0f, 3); // == 6.0
```

### 从花括号初始化列表进行 auto 推导的新规则
在使用统一初始化语法时对 `auto` 推导的修改。以前 `auto x {3};` 推导为 `std::initializer_list<int>`，现在则推导为 `int`。
```c++
auto x1 {1, 2, 3}; // 错误：不是单个元素
auto x2 = {1, 2, 3}; // x2 是 std::initializer_list<int>
auto x3 {3}; // x3 是 int
auto x4 {3.0}; // x4 是 double
```

### constexpr lambda
使用 `constexpr` 的编译期 lambda。
```c++
auto identity = [](int n) constexpr { return n; };
static_assert(identity(123) == 123);
```
```c++
constexpr auto add = [](int x, int y) {
  auto L = [=] { return x; };
  auto R = [=] { return y; };
  return [=] { return L() + R(); };
};

static_assert(add(1, 2)() == 3);
```
```c++
constexpr int addOne(int n) {
  return [n] { return n + 1; }();
}

static_assert(addOne(1) == 2);
```

### 按值捕获 `this` 的 lambda
以前在 lambda 的环境中捕获 `this` 只能是按引用捕获。一个会出问题的例子是使用回调的异步代码：它要求对象在（可能超出其生命周期的）某个时刻仍然可用。`*this`（C++17）现在会复制当前对象，而 `this`（C++11）则继续按引用捕获。
```c++
struct MyObj {
  int value {123};
  auto getValueCopy() {
    return [*this] { return value; };
  }
  auto getValueRef() {
    return [this] { return value; };
  }
};
MyObj mo;
auto valueCopy = mo.getValueCopy();
auto valueRef = mo.getValueRef();
mo.value = 321;
valueCopy(); // 123
valueRef(); // 321
```

### 内联变量
inline 说明符不仅可以用于函数，也可以用于变量。声明为内联的变量与声明为内联的函数具有相同的语义。
```c++
// 使用 compiler explorer 的反汇编示例。
struct S { int x; };
inline S x1 = S{321}; // mov esi, dword ptr [x1]
                      // x1: .long 321

S x2 = S{123};        // mov eax, dword ptr [.L_ZZ4mainE2x2]
                      // mov dword ptr [rbp - 8], eax
                      // .L_ZZ4mainE2x2: .long 123
```

它还可以用于声明并定义一个静态成员变量，这样就不需要在源文件中初始化它了。
```c++
struct S {
  S() : id{count++} {}
  ~S() { count--; }
  int id;
  static inline int count{0}; // 在类内声明并将 count 初始化为 0
};
```

### 嵌套命名空间
使用命名空间解析运算符来创建嵌套的命名空间定义。
```c++
namespace A {
  namespace B {
    namespace C {
      int i;
    }
  }
}
```

上面的代码可以写成这样：
```c++
namespace A::B::C {
  int i;
}
```

### 结构化绑定
一项关于解构初始化的提案，它允许编写 `auto [ x, y, z ] = expr;`，其中 `expr` 的类型是类元组（tuple-like）对象，其元素会被绑定到变量 `x`、`y`、`z`（由该构造声明）。*类元组对象*包括 [`std::tuple`](README.md#tuples)、`std::pair`、[`std::array`](README.md#stdarray) 以及聚合结构体。
```c++
using Coordinate = std::pair<int, int>;
Coordinate origin() {
  return Coordinate{0, 0};
}

const auto [ x, y ] = origin();
x; // == 0
y; // == 0
```
```c++
std::unordered_map<std::string, int> mapping {
  {"a", 1},
  {"b", 2},
  {"c", 3}
};

// 按引用解构。
for (const auto& [key, value] : mapping) {
  // 对 key 和 value 做一些处理
}
```

### 带初始化器的选择语句
`if` 和 `switch` 语句的新版本，它们简化了常见的代码模式，并帮助用户保持作用域紧凑。
```c++
{
  std::lock_guard<std::mutex> lk(mx);
  if (v.empty()) v.push_back(val);
}
// 对比
if (std::lock_guard<std::mutex> lk(mx); v.empty()) {
  v.push_back(val);
}
```
```c++
Foo gadget(args);
switch (auto s = gadget.status()) {
  case OK: gadget.zip(); break;
  case Bad: throw BadFoo(s.message());
}
// 对比
switch (Foo gadget(args); auto s = gadget.status()) {
  case OK: gadget.zip(); break;
  case Bad: throw BadFoo(s.message());
}
```

### constexpr if
编写根据编译期条件来决定是否实例化的代码。
```c++
template <typename T>
constexpr bool isIntegral() {
  if constexpr (std::is_integral<T>::value) {
    return true;
  } else {
    return false;
  }
}
static_assert(isIntegral<int>() == true);
static_assert(isIntegral<char>() == true);
static_assert(isIntegral<double>() == false);
struct S {};
static_assert(isIntegral<S>() == false);
```

### UTF-8 字符字面量
以 `u8` 开头的字符字面量是 `char` 类型的字符字面量。UTF-8 字符字面量的值等于其 ISO 10646 码点值。
```c++
char x = u8'x';
```

### 枚举的直接列表初始化
枚举现在可以使用花括号语法来初始化。
```c++
enum byte : unsigned char {};
byte b {0}; // OK
byte c {-1}; // 错误
byte d = byte{1}; // OK
byte e = byte{256}; // 错误
```

### \[\[fallthrough\]\]、\[\[nodiscard\]\]、\[\[maybe_unused\]\] 属性
C++17 引入了三个新属性：`[[fallthrough]]`、`[[nodiscard]]` 和 `[[maybe_unused]]`。
* `[[fallthrough]]` 向编译器表明 switch 语句中的贯穿（fall through）是有意为之的行为。该属性只能用在 switch 语句中，并且必须放在下一个 case/default 标签之前。
```c++
switch (n) {
  case 1: 
    // ...
    [[fallthrough]];
  case 2:
    // ...
    break;
  case 3:
    // ...
    [[fallthrough]];
  default:
    // ...
}
```

* 当函数或类带有 `[[nodiscard]]` 属性而其返回值被丢弃时，会发出警告。
```c++
[[nodiscard]] bool do_something() {
  return is_success; // 成功时为 true，失败时为 false
}

do_something(); // 警告：忽略了 'bool do_something()' 的返回值，
                // 该函数声明了 'nodiscard' 属性
```
```c++
// 仅当 `error_info` 按值返回时才发出警告。
struct [[nodiscard]] error_info {
  // ...
};

error_info do_something() {
  error_info ei;
  // ...
  return ei;
}

do_something(); // 警告：忽略了类型 'error_info' 的返回值，
                // 该类型声明了 'nodiscard' 属性
```

* `[[maybe_unused]]` 向编译器表明某个变量或参数可能未被使用，这是有意为之的。
```c++
void my_callback(std::string msg, [[maybe_unused]] bool error) {
  // 不关心 `msg` 是否是一条错误消息，直接记录即可。
  log(msg);
}
```

### `__has_include`

`__has_include (operand)` 运算符可以用在 `#if` 和 `#elif` 表达式中，用来检查某个头文件或源文件（`operand`）是否可用于包含。

它的一个用例是：使用两个功能相同的库，当系统上找不到首选的那个时，就使用备用的/实验性的那个。

```c++
#ifdef __has_include
#  if __has_include(<optional>)
#    include <optional>
#    define have_optional 1
#  elif __has_include(<experimental/optional>)
#    include <experimental/optional>
#    define have_optional 1
#    define experimental_optional
#  else
#    define have_optional 0
#  endif
#endif
```

它还可以用来在不知道程序运行于哪个平台的情况下，包含在不同平台上以不同名称或位置存在的头文件。OpenGL 的头文件就是一个很好的例子：在 macOS 上它们位于 `OpenGL\` 目录，而在其他平台上位于 `GL\` 目录。

```c++
#ifdef __has_include
#  if __has_include(<OpenGL/gl.h>)
#    include <OpenGL/gl.h>
#    include <OpenGL/glu.h>
#  elif __has_include(<GL/gl.h>)
#    include <GL/gl.h>
#    include <GL/glu.h>
#  else
#    error No suitable OpenGL headers found.
# endif
#endif
```

### 类模板实参推导（CTAD）
*类模板实参推导*（CTAD）允许编译器从构造函数的实参中推导模板实参。
```c++
std::vector v{ 1, 2, 3 }; // 推导为 std::vector<int>

std::mutex mtx;
auto lck = std::lock_guard{ mtx }; // 推导为 std::lock_guard<std::mutex>

auto p = new std::pair{ 1.0, 2.0 }; // 推导为 std::pair<double, double>*
```

对于用户定义的类型，如果适用，可以使用*推导指引*（deduction guide）来指导编译器如何推导模板实参：
```c++
template <typename T>
struct container {
  container(T t) {}

  template <typename Iter>
  container(Iter beg, Iter end);
};

// 推导指引
template <typename Iter>
container(Iter b, Iter e) -> container<typename std::iterator_traits<Iter>::value_type>;

container a{ 7 }; // OK：推导为 container<int>

std::vector<double> v{ 1.0, 2.0, 3.0 };
auto b = container{ v.begin(), v.end() }; // OK：推导为 container<double>

container c{ 5, 6 }; // 错误：std::iterator_traits<int>::value_type 不是一种类型
```

## C++17 库特性

### std::variant
类模板 `std::variant` 表示一个类型安全的 `union`。`std::variant` 的实例在任意时刻持有其备选类型之一的值（也可能处于无值状态）。
```c++
std::variant<int, double> v{ 12 };
std::get<int>(v); // == 12
std::get<0>(v); // == 12
v = 12.0;
std::get<double>(v); // == 12.0
std::get<1>(v); // == 12.0
```

### std::optional
类模板 `std::optional` 管理一个可选的所含值，即一个可能存在也可能不存在的值。optional 的一个常见用例是可能失败的函数的返回值。
```c++
std::optional<std::string> create(bool b) {
  if (b) {
    return "Godzilla";
  } else {
    return {};
  }
}

create(false).value_or("empty"); // == "empty"
create(true).value(); // == "Godzilla"
// 返回 optional 的工厂函数可用作 while 和 if 的条件
if (auto str = create(true)) {
  // ...
}
```

### std::any
一个类型安全的容器，可以存放任意类型的单个值。
```c++
std::any x {5};
x.has_value() // == true
std::any_cast<int>(x) // == 5
std::any_cast<int&>(x) = 10;
std::any_cast<int>(x) // == 10
```

### std::string_view
对字符串的非拥有引用。在为字符串提供抽象层（例如用于解析）时很有用。
```c++
// 普通字符串。
std::string_view cppstr {"foo"};
// 宽字符串。
std::wstring_view wcstr_v {L"baz"};
// 字符数组。
char array[3] = {'b', 'a', 'r'};
std::string_view array_v(array, std::size(array));
```
```c++
std::string str {"   trim me"};
std::string_view v {str};
v.remove_prefix(std::min(v.find_first_not_of(" "), v.size()));
str; //  == "   trim me"
v; // == "trim me"
```

### std::invoke
使用参数调用一个 `Callable` 对象。*可调用*对象的例子有 `std::function` 或 lambda；也就是可以像普通函数一样被调用的对象。
```c++
template <typename Callable>
class Proxy {
  Callable c_;

public:
  Proxy(Callable c) : c_{ std::move(c) } {}

  template <typename... Args>
  decltype(auto) operator()(Args&&... args) {
    // ...
    return std::invoke(c_, std::forward<Args>(args)...);
  }
};

const auto add = [](int x, int y) { return x + y; };
Proxy p{ add };
p(1, 2); // == 3
```

### std::apply
使用一个参数元组调用一个 `Callable` 对象。
```c++
auto add = [](int x, int y) {
  return x + y;
};
std::apply(add, std::make_tuple(1, 2)); // == 3
```

### std::filesystem
新的 `std::filesystem` 库提供了一种操作文件系统中文件、目录和路径的标准方式。

在下面的例子中，如果有可用空间，就把一个大文件复制到临时路径：
```c++
const auto bigFilePath {"bigFileToCopy"};
if (std::filesystem::exists(bigFilePath)) {
  const auto bigFileSize {std::filesystem::file_size(bigFilePath)};
  std::filesystem::path tmpPath {"/tmp"};
  if (std::filesystem::space(tmpPath).available > bigFileSize) {
    std::filesystem::create_directory(tmpPath.append("example"));
    std::filesystem::copy_file(bigFilePath, tmpPath.append("newFile"));
  }
}
```

### std::byte
新的 `std::byte` 类型提供了一种将数据表示为字节的标准方式。与 `char` 或 `unsigned char` 相比，使用 `std::byte` 的好处在于它既不是字符类型，也不是算术类型；唯一可用的运算符重载是按位运算。
```c++
std::byte a {0};
std::byte b {0xFF};
int i = std::to_integer<int>(b); // 0xFF
std::byte c = a & b;
int j = std::to_integer<int>(c); // 0
```
注意，`std::byte` 实际上就是一个枚举，而枚举的花括号初始化之所以可行，要归功于[枚举的直接列表初始化](#枚举的直接列表初始化)。

### map 和 set 的拼接（splicing）
在不产生昂贵的复制、移动或堆分配/释放开销的情况下移动节点和合并容器。

从一个 map 移动元素到另一个 map：
```c++
std::map<int, string> src {{1, "one"}, {2, "two"}, {3, "buckle my shoe"}};
std::map<int, string> dst {{3, "three"}};
dst.insert(src.extract(src.find(1))); // 低成本地从 `src` 移除并插入 { 1, "one" } 到 `dst`。
dst.insert(src.extract(2)); // 低成本地从 `src` 移除并插入 { 2, "two" } 到 `dst`。
// dst == { { 1, "one" }, { 2, "two" }, { 3, "three" } };
```

插入整个 set：
```c++
std::set<int> src {1, 3, 5};
std::set<int> dst {2, 4, 5};
dst.merge(src);
// src == { 5 }
// dst == { 1, 2, 3, 4, 5 }
```

插入生命周期长于容器的元素：
```c++
auto elementFactory() {
  std::set<...> s;
  s.emplace(...);
  return s.extract(s.begin());
}
s2.insert(elementFactory());
```

修改 map 元素的键：
```c++
std::map<int, string> m {{1, "one"}, {2, "two"}, {3, "three"}};
auto e = m.extract(2);
e.key() = 4;
m.insert(std::move(e));
// m == { { 1, "one" }, { 3, "three" }, { 4, "two" } }
```

### 并行算法
许多 STL 算法（例如 `copy`、`find` 和 `sort`）开始支持*并行执行策略*：`seq`、`par` 和 `par_unseq`，分别对应“顺序”、“并行”和“并行非顺序”。
```c++
std::vector<int> longVector;
// 使用并行执行策略查找元素
auto result1 = std::find(std::execution::par, std::begin(longVector), std::end(longVector), 2);
// 使用顺序执行策略对元素排序
auto result2 = std::sort(std::execution::seq, std::begin(longVector), std::end(longVector));
```

### std::sample
从给定序列中（无放回地）抽取 n 个元素，每个元素被选中的概率相同。
```c++
const std::string ALLOWED_CHARS = "abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789";
std::string guid;
// 从 ALLOWED_CHARS 中抽取 5 个字符。
std::sample(ALLOWED_CHARS.begin(), ALLOWED_CHARS.end(), std::back_inserter(guid),
  5, std::mt19937{ std::random_device{}() });

std::cout << guid; // 例如 G1fW2
```

### std::clamp
将给定值限制在上下界之间。
```c++
std::clamp(42, -1, 1); // == 1
std::clamp(-42, -1, 1); // == -1
std::clamp(0, -1, 1); // == 0

// `std::clamp` 也接受自定义比较器：
std::clamp(0, -1, 1, std::less<>{}); // == 0
```

### std::reduce
对给定范围内的元素进行折叠。概念上与 `std::accumulate` 类似，但 `std::reduce` 会并行地执行折叠。由于折叠是并行进行的，如果你指定了二元运算，它必须满足结合律和交换律。给定的二元运算也不应修改任何元素，或使给定范围内的任何迭代器失效。

默认的二元运算是 std::plus，初始值为 0。
```c++
const std::array<int, 3> a{ 1, 2, 3 };
std::reduce(std::cbegin(a), std::cend(a)); // == 6
// 使用自定义二元运算：
std::reduce(std::cbegin(a), std::cend(a), 1, std::multiplies<>{}); // == 6
```
此外，你还可以为归约器指定变换操作：
```c++
std::transform_reduce(std::cbegin(a), std::cend(a), 0, std::plus<>{}, times_ten); // == 60

const std::array<int, 3> b{ 1, 2, 3 };
const auto product_times_ten = [](const auto a, const auto b) { return a * b * 10; };

std::transform_reduce(std::cbegin(a), std::cend(a), std::cbegin(b), 0, std::plus<>{}, product_times_ten); // == 140
```

### 前缀和算法
支持前缀和（包括包含式扫描和排除式扫描）以及相应的变换版本。
```c++
const std::array<int, 3> a{ 1, 2, 3 };

std::inclusive_scan(std::cbegin(a), std::cend(a),
    std::ostream_iterator<int>{ std::cout, " " }, std::plus<>{}); // 1 3 6

std::exclusive_scan(std::cbegin(a), std::cend(a),
    std::ostream_iterator<int>{ std::cout, " " }, 0, std::plus<>{}); // 0 1 3

const auto times_ten = [](const auto n) { return n * 10; };

std::transform_inclusive_scan(std::cbegin(a), std::cend(a),
    std::ostream_iterator<int>{ std::cout, " " }, std::plus<>{}, times_ten); // 10 30 60

std::transform_exclusive_scan(std::cbegin(a), std::cend(a),
    std::ostream_iterator<int>{ std::cout, " " }, 0, std::plus<>{}, times_ten); // 0 10 30
```

### GCD 和 LCM
最大公约数（GCD）和最小公倍数（LCM）。
```c++
const int p = 9;
const int q = 3;
std::gcd(p, q); // == 3
std::lcm(p, q); // == 9
```

### std::not_fn
返回给定函数结果取反的工具函数。
```c++
const std::ostream_iterator<int> ostream_it{ std::cout, " " };
const auto is_even = [](const auto n) { return n % 2 == 0; };
std::vector<int> v{ 0, 1, 2, 3, 4 };

// 打印所有偶数。
std::copy_if(std::cbegin(v), std::cend(v), ostream_it, is_even); // 0 2 4
// 打印所有奇数（非偶数）。
std::copy_if(std::cbegin(v), std::cend(v), ostream_it, std::not_fn(is_even)); // 1 3
```

### 字符串与数字之间的转换
在字符串与整型/浮点型之间进行转换。这些转换不会抛出异常，不会分配内存，并且比 C 标准库中的等价函数更安全。

对于 `std::to_chars`，用户需要负责分配所需的足够存储空间，否则函数会失败并在其返回值中设置错误码对象。

这些函数允许你可选地传入一个进制（默认为十进制），或者为浮点类型输入传入一个格式说明符。

* `std::to_chars` 返回一个（非 const 的）char 指针，指向函数在给定缓冲区内写入的字符串末尾的下一个位置，以及一个错误码对象。
* `std::from_chars` 返回一个 const char 指针，成功时它等于传给函数的结束指针，以及一个错误码对象。

这两个函数返回的错误码对象在成功时都等于默认初始化的错误码对象。

将数字 `123` 转换为 `std::string`：
```c++
const int n = 123;

// 可以使用任意容器、字符串、数组等。
std::string str;
str.resize(3); // 为 `n` 的每一位数字预留足够的存储空间

const auto [ ptr, ec ] = std::to_chars(str.data(), str.data() + str.size(), n);

if (ec == std::errc{}) { std::cout << str << std::endl; } // 123
else { /* 处理失败 */ }
```

将值为 `"123"` 的 `std::string` 转换为整数：
```c++
const std::string str{ "123" };
int n;

const auto [ ptr, ec ] = std::from_chars(str.data(), str.data() + str.size(), n);

if (ec == std::errc{}) { std::cout << n << std::endl; } // 123
else { /* 处理失败 */ }
```

### 用于 chrono duration 和 timepoint 的取整函数
为 `std::chrono::duration` 和 `std::chrono::time_point` 提供 abs、round、ceil 和 floor 辅助函数。
```c++
std::chrono::milliseconds a{ -5500 };
std::chrono::milliseconds d = std::chrono::abs(a); // == 5500ms
std::chrono::round<seconds>(d); // == 6s
std::chrono::ceil<seconds>(d); // == 6s
std::chrono::floor<seconds>(d); // == 5s
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
