---
description: Effective Modern C++ 读书笔记
---

# Effective Modern C++

## 第一章 类型推导

### 条款1 理解类型推导

```cpp
template<typename T>
void f(ParamType param)
# 调用
f(expr)
```

`T` 和 `ParamType` 很多时候推导出来是不一样的，因为 `ParamType` 常常带有修饰，比如 `const` 、`&` 等。

```cpp
template<typename T>
void f(const T& param)
# 调用
int x = 0;
f(x);
```

`T` 推导为 `int` ， `ParamType` 推导为 `const int&` 。

**因此 T 的类型推导不仅依赖于 expr 的类型，还依赖于 ParamType 的形式。**

#### case 1：ParamType是引用或指针，但不是通用引用\(Universal Reference\)

1. 如果 `expr` 是引用，忽略引用部分
2. 按照模式匹配的规则根据 `ParamType` 推导 `T` 

#### case 2：ParamType是通用引用

* 如果 `expr` 是左值，`T` 和 `ParamType` 推导为左值引用
  * 这是 T 被推导为引用的唯一一种情况
  * 虽然 `ParamType` 声明语法跟右值引用相同，但其被推导为左值引用
* 如果 `expr` 是右值，根据 case 1 规则推导

#### case 3：ParamType既不是引用也不是指针

1. 如果 `expr` 是引用，忽略引用部分
2. 忽略引用之后，如果 `expr` 由 const / volatile 修饰，将其忽略

```cpp
template<typename T>
void f(T param)
# 调用
const char* const ptr =    // ptr is const pointer to const object
    "Fun with pointers";
f(ptr);                    // type deduced const char*
```

#### 数组参数

* 数组按值传递时，T 推导为 const type\* （指针）
* 数组按引用传递时，T 推导为 const type \(&\)\[N\] （数组），N 为数组大小

```cpp
# 可在编译期获得数组大小
template<typename T, std::size_t N>
constexpr std::size_t arraySize(T (&)[N]) noexcept
{
    return N;
}
```

#### 函数参数

* 函数按值传递时，T 推导为函数指针
* 函数按引用传递时，T 推导为函数引用

### 条款2 理解auto类型推导

* 将 `auto` 看作 条款1 中的 T，type specifier 看作 ParamType，规则同样适用
* auto 类型推导 与 模板类型推导 的区别在于：auto 将 `braced initializer` 推导为 `std::initializer_list<T>` ，模板类型推导则会拒绝这种类型

  ```cpp
  auto x = {11, 23, 9}; // x's type is std::initializer_list<int>

  template<typename T>
  void f(T param);
  f({11, 23, 9});   // can't deduce type for T

  template<typename T>
  void f(std::initializer_list<T> param);
  f({11, 23, 9});   // T's type is int
  ```

* C++ 14 中 auto 作为函数返回值类型时，按照模板类型推导规则进行

### 条款3 理解 decltype

* `decltype` 准确地返回传递给它的对象类型

  ```cpp
  // 此时 auto 不做类型推导，而表示尾置返回类型
  template<typename Container, typename Index> 
  auto authAndAccess(Container& c, Index i) 
      -> decltype(c[i]) // refinement
  {
      authenticateUser();
      return c[i];
  }
  ```

* `decltype(auto)` 表示根据 decltype 规则进行类型推导
* For lvalue expressions of type T other than names, decltype always reports a type of T&.
* C++14 supports decltype\(auto\), which, like auto, deduces a type from its initializer, but it performs the type deduction using the decltype rules.

### 条款4 如何查看推导类型

* IDE 自带类型推导功能
* 编译器诊断
* 运行时输出，标准库\(不一定正确\) / boost库\(√\)

## 第二章 auto

### 条款5 尽量使用auto，而不是显式声明

### 条款6 Use the explicitly typed initializer idiom when auto deduces undesired types

## 第三章 Modern C++

### 条款7 构建对象时区分\(\)和{}

* 花括号初始化时使用范围最广的初始化语法，它能避免 narrowing conversions，不会产生 most vexing parse

  ```cpp
  double x, y, z;
  int sum{x + y + z}; // error! narrowing conversion
  int sum(x + y + z); // OK, double is truncated to int

  Widget w1(); // function declaration
  Widget w2{}; // no args initialization
  ```

* 构造函数重载解析时，花括号初始化会优先匹配参数为 std::initializer\_list 的构造函数（只要类型可以隐式转换，都优先匹配）。
* 典型例子

  ```cpp
  vector<int> v(10, 20); // 10 elements, value 20
  vector<int> v{10, 20}; // 2 elements, value 10 and 20
  ```

### 条款8 尽量用 nullptr，而不是 0 和 NULL

* 0 和 NULL 都不是指针类型，因此作为参数传递时不会调用以指针为参数的重载
* nullptr 实际类型为 std::nullptr\_t ，既不是数值类型也不是指针类型，但是你可以将其视作任意类型的指针

### 条款9 尽量用 alias declaration，而不是 typedef

* typedef 不支持模板化，alias declaration 支持

  ```cpp
  template<typename T> // MyAllocList<T>
  using MyAllocList = std::list<T, MyAlloc<T>>; // is synonym for std::list<T, MyAlloc<T>>
  MyAllocList<Widget> lw; // client cod

  // typedef 需要用结构体包裹
  template<typename T> // MyAllocList<T>::type
  struct MyAllocList { // is synonym for
      typedef std::list<T, MyAlloc<T>> type; // std::list<T, MyAlloc<T>>
  }; 
  MyAllocList<Widget>::type lw; // client code
  ```

### 条款10 尽量用 scoped enums，而不是 unscoped enums

```cpp
/* unscoped enums */
enum Color {black, white, red};
auto white = false;  // error

/* scoped enums */
enum class Color {black, white, red};
auto white = false;      // fine
Color c = white;         // error
Color c = Color::white;  // fine
```

* scoped enums 可以减少命名空间污染
* unscoped enums 有类型隐式转换的优势，默认是整型，并可以隐式转换为其他数值类型；scoped enums 则不允许隐式转换，可以显式转换
* scoped enums 允许前向声明，unscoped enums 只有制定了数据类型才能前向声明
* 都支持指定使用的数据类型，scoped enums 默认使用 int，unscoped enums 没有默认类型

### 条款11 使用 delete 函数，而不是 private 未定义函数

* delete 函数一般声明为 public，因为权限检查先于 delete 状态检查。如果声明为 private ，编译期报错是关于 private 的，使得报错信息模糊。
* delete 可以修饰 **任何** 函数

  ```cpp
  // 防止隐式转换
  bool isLucky(int num);
  bool isLucky(char) = delete;
  bool isLucky(bool) = delete;
  // 也会拒绝 float 参数，C++倾向于将float转换为double
  bool isLucky(double) = delete;
  ```

* delete 还可用于禁用部分模板实例化

  ```cpp
  template<typename T>
  void processPointer(T* ptr);

  template<>
  void processPointer<void>(void*) = delete;
  template<>
  void processPointer<char>(char*) = delete;
  ```

* 对于类内模板，模板实例化必须写在命名空间域内，而不是类域内。所以无法通过声明private 并 不定义 来禁用类内模板实例。应该使用 delete。

### 条款12 将 overriding function 声明为 override

* 在派生类中覆写的虚函数声明为 override，便于编译器查错
* reference qualifiers 可以区别对待左值对象和右值对象

  ```cpp
  class Widget {
  public:
      using DataType = std::vector<double>;
      …
      DataType& data() & // for lvalue Widgets, return lvalue
      { return values; } 
      DataType data() && // for rvalue Widgets, return rvalue
      { return std::move(values); } 
      …
  private:
      DataType values;
  };

  Widget makeWidget(); // factory function

  // calls lvalue overload for Widget::data, copy constructs vals1
  auto vals1 = w.data(); 
  // calls rvalue overload for Widget::data, move constructs vals2
  auto vals2 = makeWidget().data();
  ```

### 条款13 尽量用 const\_iterator，而不是 iterator

* 尽量用 const\_iterator，而不是 iterator

  ```cpp
  // C++ 11 未提供 cbegin cend rbegin rend crbegin crend 等 free function
  template<class C>
  auto cbegin(const C& container) -> decltype(std::begin(container))
  {
      return std::begin(container);
  }
  ```

* 为了代码的泛用性，最好使用非成员函数的 begin / end / cbegin / cend 等

### 条款14 如果函数不会抛出异常，声明为 nonexcept

### 条款15 尽可能使用 constexpr

* constexpr 对象都是 const 的，且被编译期值初始化
* 当函数被声明为 constexpr 时，若传入的参数是编译期可以获知的，那它可以产生编译期值；若传入参数是运行时获知，那它跟普通函数行为一致
* 在 C++ 11 中，constexpr 函数只能含有一条 return 语句（可以用 “？：” 代替 if else，用递归代替循环）；C++ 14 中没有此限制
* constexpr 函数只能接受和返回常量类型（literal types）。C++ 11 中，除了 void 之外的所有 built-in 类型都符合条件。用户自定义类型也可以是常量类型，因为构造函数和其他成员函数可以是 constexpr。
* C++ 11 中，constexpr 成员函数隐式定义为 const（不能修改成员值），且不支持 void 作为返回类型，所以 setter 函数不能被声明为 constexpr。C++ 14 中无此限制。

### 条款16 确保 const 成员函数线程安全

* 确保 const 成员函数线程安全，除非你能确保它们永远不会被并发调用
* std::atomic 变量可能可以提供比 mutex 更好的性能，但是它只适用于只对一个变量或内存位置进行操作的情况

### 条款17 理解特殊成员函数的生成

## 第四章 智能指针

### 条款18 使用 std::unique\_ptr 进行独有资源管理

* std::unique\_ptr 应用的典型场景就是工厂函数返回对象
* std::unique\_ptr 是 `move-only` 的
* 析构时默认调用 delete，也可以自定义\(custom deleters\)。推荐使用无捕获的 lambda 表达式，因为它不会增加智能指针的体积；用函数指针至少会导致 std::unique\_ptr 增加函数指针的大小。
* std::unique\_ptr 很容易转换为 std::shared\_ptr

### 条款19 使用 std::shared\_ptr 进行共享资源管理

* std::shared\_ptr 是原始指针的两倍大小。包含一个指向资源的 raw pointer 和 一个指向引用计数的 raw pointer
* 引用计数的内存必须动态分配
* 引用计数的增减必须是原子的。你应当认为原子操作的代价是相当昂贵的
* 移动构造 比 拷贝构造 更加高效，前者不改变引用计数
* std::unique\_ptr 的自定义析构函数是 自身的一部分 会改变其类型/大小，std::shared\_ptr 则不会，而是将其存放在 共享的控制块\(control block\) 中。

  ```cpp
  std::unique_ptr<
      Widget, decltype(loggingDel)
  > upw(new Widget, loggingDel);

  std::shared_ptr<Widget>
      spw(new Widget, loggingDel);
  ```

  ![](../.gitbook/assets/shared-ptr-structure.png)

* 控制块的构建
  * std::make\_shared 总是创建一个控制块
  * 当 std::shared\_ptr 由独有权限指针\(比如 std::unique\_ptr\)构建时，创建控制块
  * 由 raw pointer 构造 std::shared\_ptr 时，创建控制块
* `std::enable_shared_from_this` ，使用 CRTP \(Curiously Recurring Template Pattern\) 设计模式。
* 避免由 raw pointer 创建 std::shared\_ptr

### 条款20 用 std::weak\_ptr 来避免空悬指针

* `expired()` 用于检测所指对象是否还存在
* `lock()` 是为了提供 检测+获取 的原子操作。如果检测和获取分开，可能会导致 race condition
* 用 std::weak\_ptr 来避免空悬指针
* 使用场景：缓存、observer lists、防止 std::shared\_ptr 循环引用
* std::weak\_ptr 与 std::shared\_ptr 大小一致，操作 控制块 中的 二级引用\(second reference\)

### 条款21 std::make\_unique / std::make\_shared 代替直接适用 new

* std::make\_shared - C++ 11，std::make\_unique - C++ 14
* 使用 make function 可以避免代码重复，减少编译时间
* make function 是异常安全的
* std::make\_shared 可以产生更高效的代码

  ```cpp
  // 需要两次内存分配，一次给 Widget，一次给控制块
  std::shared_ptr<Widget> spw(new Widget);
  // 一次内存分配，Widget和控制块一起保存
  auto spw = std::make_shared<Widget>();
  ```

* make function 的限制，std::unique 只有前两条限制，std::shared\_ptr 限制条件更多
  * make function 不允许自定义析构函数
  * make function 不支持花括号初始化，需要借助 auto 推导实现

    ```cpp
    // create std::initializer_list
    auto initList = { 10, 20 };
    // create std::vector using std::initializer_list ctor
    auto spv = std::make_shared<std::vector<int>>(initList);
    ```

  * 由于 make function 分配一块内存共同持有对象和控制块，因此，只有当所有 shared\_ptr 和 weak\_ptr 都失效时，内存才会被释放；仅所有 shared\_ptr 失效时，对象不会被释放。所以当内存吃紧时，酌情使用
  * 当需要自定义内存管理（分配/析构）时，无法使用 make\_shared

### 条款22 When using the Pimpl Idiom, define special member functions in the implementation file.

* `Pimpl`  - pointer to implementation
* Pimpl Idiom 是一种减少编译期依赖的方法，可以减少编译时间
* 对于 std::unique\_ptr ，建议将特殊函数声明/实现分离，即使默认生成函数就满足要求，也建议将其显式声明，在 .cc/.cpp 中实现。
* 上述建议不适用与 std::shared\_ptr，因为编译器生成特殊函数时，不要求 shared\_ptr\(自定义内存分配函数不是对象的一部分\) 指向的对象是 complete

## 第五章 右值引用、移动语义、完美转发

### 条款23 std::move 和 std::forward

* std::move 和 std::forward 在运行时不产生可执行代码，它们仅做类型转换
  * std::move 无条件进行右值转换
  * std::forward 执行有条件的类型转换，当且仅当输入参数被右值初始化\(绑定\)时，将其转换为右值。典型场景：函数模板接受通用引用\(Universal reference\)参数并传递给另一个函数。

    > 函数的参数作为参数传递给其内部的函数时，是一个左值，若想维持其类型不变，就用 std::forward
* 如果你想移动对象，不要用 `const` 修饰它。

### 条款24 区分 universal references 和 rvalue references

```cpp
template<class T>
void f(T&& param);  // universal reference

auto&& var1 = var2; // universal reference
```

* universal reference \(forwarding reference\) 必须严格遵守 `type&&` 形式，但是如果没有发生类型推导，`type&&` 就表示右值引用
* universal reference 既可绑定左值，也可绑定右值

### 条款25 Use std::move on rvalue references, std::forward on universal references

* 在最后一次使用变量时使用 std::move / std::forward 
* 对于 return by value 的函数，将右值引用和universal reference做相同处理。（std::forward）????
* 除非局部对象不适合 RVO \(return value optimization\) ，否则不要对其使用 std::move / std::forward 

### 条款26 Avoid overloading on universal references

### 条款28 理解引用折叠 \(reference collapsing\)

### 条款29 Assume that move operations are not present, not cheap, and not used

### 条款30 熟悉完美转发的失效情形

## 第六章 Lambda

[lambda表达式基本语法](https://docs.microsoft.com/en-us/cpp/cpp/lambda-expressions-in-cpp?view=vs-2019%20)

```cpp
[captured param](param list) mutable throw() -> type { expressions... }
```

### 条款31 避免默认捕捉模式

* 默认引用捕捉模式时，若 lambda 生命周期长于局部变量，会造成空悬引用
* 默认按值捕捉模式时，可能会导致空悬指针 / 对于static storage duration 变量，会让人误解其为self-contained

  ```cpp
  class Widget
  {
  public:
      void addFilter() const;
  private:
      int divisor;
  }

  void Widget::addFilter() const
  {
      // 实际上复制了this指针，导致this指针泄露
      filter.emplace_back(
          [=](int value){return value % divisor == 0;});
  }

  void Widget::addFilter() const
  {
      // error 不存在局部变量 divisor
      filter.emplace_back(
          [divisor](int value){return value % divisor == 0;});
  }

  void Widget::addFilter() const
  {
      // error divisor 不可得
      filter.emplace_back(
          [](int value){return value % divisor == 0;});
  }

  void Widget::addFilter() const
  {
      // 看似按值捕捉，实际是直接使用的divisor，所有lambda内的值都会随着divisor变化而变化
      static int divisor = 1;
      filter.emplace_back(
          [=](int value){return value % divisor == 0;});
      ++divisor;
  }
  ```

### 条款32 Use init capture to move objects into closures

* C++ 14 使用 init capture 更好

  ```cpp
  auto func = [pw = std::make_unique<Widget>()](){
      return pw->isValidated() && pw->isArchived();
  }
  ```

* C++ 11 可以用 std::bind 模拟 init capture

  ```cpp
  auto func = std::bind(
      [](const std::unique_ptr<Widget>& pw)
      { return pw->isValidated() && pw->isArchived(); },
      std::make_unique<Widget>());
  ```

### 条款33 Use decltype on auto&& parameters to std::forward them

* C++ 14 支持 generic lambda

  ```cpp
  // 有限参数
  auto f = [](auto&& param) { 
      return func(normalize(std::forward<decltype(param)>(param)); }
  // 可变参数              
  auto f = [](auto&&... param) { 
      return func(normalize(std::forward<decltype(param)>(param)...); }
  ```

### 条款34 尽量使用 lambda，而不是 std::bind

* lambda 更加易读，表达能力更强（比如能区分重载函数），更可能比 std::bind 高效
* 在 C++ 11 中，std::bind 对于模拟 move capture 以及 绑定模板函数调用操作符 时有用

## 第七章 并发API

### 条款35 Prefer task-based programming to thread-based

* std::thread 不提供直接获取异步运行函数返回值的手段，且抛出异常时，程序终止。
* 并发C++程序中“线程”的三种含义
  * 硬件线程：实际执行计算的线程。当前的计算机架构每核支持一个或多个线程。
  * 软件线程（OS线程/系统线程）：由操作系统管理/调度在硬件线程上执行的线程。软件线程可以多于硬件线程。**有限资源** ，试图创建超过系统承载能力的软件线程会抛出 std::system\_error 异常，即使声明 noexcept 也会抛出。
  * std::thread：C++进程中的对象，作为操作底层软件线程的 handle。std::thread 可以是 “null” handle，即不控制任何软件线程。
* oversubscription 问题：ready-to-run 的软件线程数多于硬件线程数
  * 导致频繁的线程切换，CPU cache 被新线程污染，保存不了多少有用的指令/数据
* std::async 不保证会创建一个新线程
* 以下情况直接使用线程合适
  * 当你需要使用底层线程API时，std::thread 提供了 native\_handle
  * 当你需要且有能力优化线程使用时
  * 当C++并发API 不支持你实现线程技术时

### 条款36 当异步很关键时，指明 std::launch::async

* 两种启动策略\(launch policy\)

  * std::launch::async：函数 f 必须异步执行，即另起线程
  * std::launch::deferred：当调用 get / wait 方法时才会异步执行 f，调用方会一直阻塞直到 f 完成；如果一直不调用 get / wait，f 就永远不会执行

  ```cpp
  // run f using default launch policy
  // 默认策略允许 f 同步/异步 运行
  auto fut1 = std::async(f);
  // run f either async or deferred
  auto fut2 = std::async(std::launch::async | std::launch::deferred,f);
  ```

* 假设线程 t 执行 std::async\(f\)
  * 不能确定 f 是否与 t 并发执行，可能被 deferred
  * 不能确定 f 是否运行在与调用 get/wait 的线程不同的线程
  * 不能确定 f 是否运行 
* 使用默认策略 std::async 的条件
  * task 不需要与调用 get/wait 的线程并发执行
  * 不会影响 thread\_local 变量的读写
  * 保证 get/wait 一定会被调用，或者 task 是否执行也无关紧要
  * 调用 wait\_for 或 wait\_until 的代码必须考虑检查 deferred 状态
* 不满足以上条件时，指定 std::launch::async

### 条款37 在任意执行路径上保证 std::thread 都是 unjoinable 的

* unjoinable 的 std::thread 包括
  * 默认构造的 std::thread
  * 移动构造时失去控制权的 std::thread
  * 已调用 join 的 std::thread
  * 已调用 detach 的 std::thread
* 析构 joinable 的线程会导致程序终止
* 在数据成员的末尾声明 std::thread 对象
* 在析构时 join / detach 都可能导致异常现象，更为合适的方法是使用可中断线程，参考C++ Concurrency in Action 9.2 节

### 条款38 认识不同的 thread handle destructor 行为

* std::future / std::promise 的结果保存于 `Shared State` （堆对象）中

  ![](../.gitbook/assets/shared-state.png)

* future 析构函数的行为
  * The destructor for the last future referring to a shared state for a nondeferred task launched via std::async blocks until the task completes
  * The destructor for all other futures simply destroys the future object
* future 析构函数正常情况下仅销毁 future 的数据成员
* 最后一个指向 shared state 的 non-deferred task\(由std::async启动\) 的 future 会阻塞，直至 task 完成

### 条款39 考虑使用 void futures 进行一次性事件通信

* 用条件变量做事件唤醒的弊端
  * 如果唤醒线程 notify 先于等待线程 wait 执行，可能会发生信号丢失
  * wait 可能会被 恶意唤醒 \(spurious wakeups\)
* 共享bool变量的弊端
  * 等待线程轮询 bool 值会造成浪费
* 联合使用条件变量和共享变量，不会存在问题，但是相当于双重检查，不够clean
* one-shot 事件推荐使用 void promise/future/shared\_future，会带来堆内存分配的开销

  ```cpp
  std::promise<void> p;
  p.get_future().wait();
  p.set_value();
  ```

### 条款40 使用 std::atomic 做并发，volatile 修饰 special memory

* std::atomic 用于多线程编程时声明原子数据及其操作，删除了拷贝构造、移动构造、移动赋值。默认为 sequential consistency
* volatile 用于提示编译不要优化读写操作，用于处理 special memory

  ```cpp
  int x;
  auto y = x;
  y = x;
  // may be optimized as
  // auto y = x;

  volatile int x;
  auto y = x; // read x
  y = x;      // read x again (can't be optimized)
  // can't be optimized
  ```

## 第八章 微调

### 条款41 对于 易于move和总是拷贝的可拷贝参数，按传值的方式传参

### 条款42 用 emplacement 代替 insertion

```cpp
std::vector<std::string> vs;
vs.push_back("xyzzy");  // wrong
vs.push_back(std::string("xyzzy"));  // right, 但是引入了两次构造和一次析构
vs.emplace_back("xyzzy");  // 直接构造容器内元素，只要一次构造
```

* emplacement 和 insertion 哪个实际更快，需要实际 benchmark 测试。但是仍然有几个条件作为启发式，当满足时，emplacement 通常更快
  * 构造插入而非赋值插入时。Node-based 的容器通常都是构造插入 。
  * 传入的参数类型与容器保存的类型不一致时
  * 容器允许重复元素
* 当容器保存智能指针时，使用 insertion 可以防止内存泄漏。因为避免了构造时异常。
* emplacement 使用直接初始化，你需要自己保证传入的参数是有效的。

  ```cpp
  regexes.emplace_back(nullptr); // compiles. but behavior undefined
  regexes.push_back(nullptr); // error
  std::regex r1 = nullptr; // error
  std::regex r2(nullptr); // compiles
  ```

