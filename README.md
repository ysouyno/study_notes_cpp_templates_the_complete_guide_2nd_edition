# study_notes_cpp_templates_the_complete_guide_2nd_edition

我想这里的笔记使用不一样的方式，尽量少写文字，多写代码，以注释的形式进行说明。

## <2022-05-24 Tue>

``` c++
// Some Remarks About Programming Style, page xxxi

#include <iostream>

typedef char* CHARS;

// 注意这里不是单纯的替换
typedef const CHARS CPTR1; // constant pointer to chars
typedef const char* CPTR2; // pointer to constant chars

int main() {
  char a = 'a', b = 'b';
  CPTR1 A = &a;
  CPTR2 B = &b;
  // 因为`A`是`const`指针，所以这里不能修改它的值
  A = &b;
  // 因为`B`是可写指针，指向不可写的`const`变量，所以这里可以修改
  B = &a;
}
```

``` c++
// 1.1.2 Using the Template, page 4

// 类型`T`这种传值的方式，则`T`必须支持拷贝构造函数，
// 但是`C++17`后，类型`T`却可以不用支持拷贝构造函数，
// 也可以传递临时值（右值），或者`T`不支持复制及移动
// 构造函数
template <typename T>
T max(T a, T b) {
  return b < a ? a : b;
}
```

``` c++
#include <iostream>
#include <type_traits>

// 1.3 Multiple Template Parameters, page 9
// 这是重要概念，看书时要特别小心
// `T` is `template parameter`
// `a` and `b` are `call parameters`
template <typename T> T max(T a, T b);

// 1.3.1 Template Paramters for Return Types, page 10
// 因为模板推导不了返回值`RT`的类型，它的位置被放在了最后，
// 所以调用的时候要指定所有参数的类型，即：
// max0<int, double, double>(4, 7.2);
template <typename T1, typename T2, typename RT> RT max0(T1 a, T2 b) {
  return b > a ? b : a;
}

// 这里将`RT`放到最前面，可以这样调用：
// max1<double>(4, 7.2);
// 这里的`double`是返回值类型，`T1`和`T2`可以被推导出来，
template <typename RT, typename T1, typename T2> RT max1(T1 a, T2 b) {
  return b > a ? b : a;
}

// 1.3.2 Deducing the Return Type, page 11
// 如果返回值依赖`template parameters`，把推导交给编译器是最简单的，
// 从`C++14`开始可以使用`auto`关键字，如下：
template <typename T1, typename T2> auto max2(T1 a, T2 b) {
  return b > a ? b : a;
}

// 1.3.2 Deducing the Return Type, page 11
// 这里有一个大缺陷就是返回值可能会是引用类型，因`T`可能是一个引用
// 所有要用`decay`，见下个函数`max3`
// `->`是`trailing return type`
template <typename T1, typename T2>
auto max3(T1 a, T2 b) -> decltype(b < a ? a : b) {
  return b < a ? a : b;
}

// 1.3.2 Deducing the Return Type, page 12
// 函数中为什么要用`true`？我的理解是将它当做一个函数声明来理解
template <typename T1, typename T2>
auto max4(T1 a, T2 b) -> typename std::decay<decltype(true ? a : b)>::type {
  return b < a ? a : b;
}

// 1.3.3 Return Type as Common Type, page 12
// 前两天看了一下`metafunction`，感觉这里的解释有点像元编程：
// `std::common_type`是一个`type trait`，它生成一个结构体，该结构体有
// 一个类型成员做为结果类型
template <typename T1, typename T2>
std::common_type_t<T1, T2> max5(T1 a, T2 b) {
  return b < a ? a : b;
}

// 1.4 Default Template Arguments, page 13
// 这里需要调用传入类型的无参构造函数，似乎有点不那么通用
template <typename T1, typename T2,
          typename RT = std::decay_t<decltype(true ? T1() : T2())>>
RT max6(T1 a, T2 b) {
  return b < a ? a : b;
}

// 1.4 Default Template Arguments, page 13
template <typename T1, typename T2, typename RT = std::common_type_t<T1, T2>>
RT max7(T1 a, T2 b) {
  return b < a ? a : b;
}

int main() {
  int a = 20;
  double b = 4.2;
  int &c = a;
  std::cout << ::max3(c, b) << '\n';

  // 1.3.2 Return Type as Common Type, page 12
  // 这里`ir`是引用类型，但是遇到`auto`后就被`decayed`了，因为：
  // "Note that an initialization of type always decays."
  int i = 42;
  int const &ir = i;
  auto ai = ir;
  std::cout << ai << '\n';

  std::cout << ::max6(4, 7.2) << '\n';
  std::cout << ::max6(7.2, 4) << '\n';
  // 因为`max7`的`RT`是最后一个参数，所以`<>`中要写满三个类型
  std::cout << ::max7<double, int, long double>(7.2, 4) << '\n';

  return 0;
}
```

## <2022-05-25 Wed>

``` c++
// 1.5 Overloading Function Templates, page 18

#include <cstring>
#include <iostream>

// call-by-reference
template <typename T> T const &max(T const &a, T const &b) {
  std::cout << "T const &(T const &, T const &)\n";
  return b < a ? a : b;
}

// call-by-value，这里为什么说是传值而不说是传指针？
// 紧挨着的下面那段代码有演示
char const *max(char const *a, char const *b) {
  std::cout << "char const *(char const *, char const *)\n";
  return std::strcmp(b, a) < 0 ? a : b;
}

template <typename T> T const &max(T const &a, T const &b, T const &c) {
  std::cout << "T const &(T const &, T const &, T const &)\n";
  return max(max(a, b), c); // error if max(a, b) uses call-by-value
}

int main() {
  // 书中问这里为什么没有遇到同样的问题？（运行时崩溃）
  // `(7, 42, 68)`参考所创建的临时值是在`main()`函数中的，
  // 它会持续存在，直到语句结束
  auto m1 = ::max(7, 42, 68);
  std::cout << m1 << '\n';

  std::cout << "-------------\n";

  char const *s1 = "frederic";
  char const *s2 = "anica";
  char const *s3 = "lucas";
  auto m2 = ::max(s1, s2, s3); // run-time ERROR
  std::cout << m2 << '\n';
}
```

``` c++
#include <cstring>
#include <iostream>
#include <stdio.h>

// 这里虽然是传指针，本质其实是传值，通过程序输出可以看出：
// -------------------------------------
// a: 0x562e5395c022, &a: 0x7ffe6df48f38
// b: 0x562e5395c028, &b: 0x7ffe6df48f40
// a: 0x562e5395c022, &a: 0x7ffe6df48f18
// b: 0x562e5395c028, &b: 0x7ffe6df48f10
// world
// -------------------------------------
// call-by-value
const char *max(const char *a, const char *b) {
  printf("a: %p, &a: %p\n", a, &a);
  printf("b: %p, &b: %p\n", b, &b);

  return std::strcmp(b, a) < 0 ? a : b;
}

int main() {
  const char *a = "hello";
  const char *b = "world";
  printf("a: %p, &a: %p\n", a, &a);
  printf("b: %p, &b: %p\n", b, &b);

  std::cout << max(a, b) << '\n';
  return 0;
}
```

``` c++
// 1.5 Overloading Function Templates, page 19

#include <iostream>

template <typename T> T max(T a, T b) {
  std::cout << "max<T>()\n";
  return b < a ? a : b;
}

// 这里只使用了`max<T>()`的版本，而没有用到`int max(int, int)`版本
// 因为`max<T>()`的定义在前，`int max(int, int)`在后，把它们都放到
// 这个函数前面，那么`int max(int, int)`这个特化版本将被调用
template <typename T> T max(T a, T b, T c) { return max(max(a, b), b); }

int max(int a, int b) {
  std::cout << "int max(int, int)\n";
  return b < a ? a : b;
}

int main() { ::max(47, 11, 33); }
```

下面代码中关于友函数那段内容没有看懂。

``` c++
#include <cassert>
#include <iostream>
#include <vector>

// 2.4 Friends, page 31
template <typename T> class Stack;
template <typename T>
std::ostream &operator<<(std::ostream &, Stack<T> const &);

template <typename T> class Stack {
private:
  std::vector<T> elems;

public:
  void push(T const &elem);
  T pop();
  T const &top() const;
  bool empty() const { return elems.empty(); }

  void print_on(std::ostream &strm) const {
    for (T const &elem : elems) {
      strm << elem << ' ';
    }
  }

  // 2.4 Friends, page 30
  // 这里不太看得懂，提供了两个选项关于如何添加友函数，但是最后这两个选项
  // 应用之后依然没有解决呀！

  // 选项1，既不能继续使用`T`，也不能跳过模板参数声明，这里使用`U`
  // 要么内部的`T`隐藏外部的`T`；要么在命名空间作用域内声明一个非模板函数
  template <typename U>
  friend std::ostream &operator<<(std::ostream &strm, Stack<U> const &s) {
    s.print_on(strm);
    return strm;
  }

  // 选项2，要使用前置声明
  // `operator<<`后跟了`<T>`，这是一个非成员函数模板的特化做为友函数
  // 原文：a specialization of the nonmember function template as friend.
  friend std::ostream &operator<<<T>(std::ostream &strm, Stack<T> const &s) {
    return strm;
  }
  // 选项2，要使用前置声明
  // `operator<<`后没有`<T>`，这是一个新的非模板函数
  // 原文：a new nontemplate function
  friend std::ostream &operator<<(std::ostream &strm, Stack<T> const &) {
    return strm;
  }
};

template <typename T> void Stack<T>::push(T const &elem) {
  elems.push_back(elem);
}

template <typename T> T const &Stack<T>::top() const {
  assert(!elems.empty());
  return elems.back();
}

template <typename T> T Stack<T>::pop() {
  assert(!elems.empty());
  T elem = elems.back();
  elems.pop_back();
  return elem;
}

int main() {
  Stack<std::pair<int, int>> ps; // note: std::pair<> has no operator<< defined
  ps.push({4, 5});
  ps.push({6, 7});
  std::cout << ps.top().first << '\n';
  std::cout << ps.top().second << '\n';
  // 仅当调用下面这个函数时才会出现编译错误，因为如果不使用的话，模板
  // 不做检查，因为`std::pair<>`没有`<<`操作，所以这里会出错
  // ps.print_on(std::cout);
}
```

看到这里才发现，第一部分全是粗略的介绍，细节全在第二部分中，难道两年前看这本书的时候就是因为这点儿才放弃的？

``` c++
#include <vector>

template <typename T> class Stack {
private:
  std::vector<T> elems;

public:
  Stack() = default;
  // Stack(T const &elem) : elems({elem}) {}
  // Stack(T elem) : elems({elem}) {}
  Stack(T elem) : elems({std::move(elem)}) {} // better
};

int main() {
  Stack int_stack = 0; // Stack<int> deduced since C++17

  // 2.9 Class Template Argument Deduction, page 41
  // 这里编译出一大堆错误，因为`Stack`的单参数构造函数是传引用
  // 传引用：参数不会`decay`，但是传值：参数可以`decay`
  // 因此这样的构造函数：Stack(T const &elem) : elems({elem}) {}
  Stack string_stack1 = "bottom"; // Stack<char const[7]> deduced since C++17
  // 这样的构造函数：Stack(T elem) : elems({elem}) {}
  Stack string_stack2 = "bottom"; // Stack<char const *> deduced since C++17
}
```

## <2022-05-29 Sun>

意思就是原来都是`T`做为参数，在调用时传的是类型，比如`int`，现在这个“非类型模板参数”就是指调用时传的是值，比如代码中的`20u`。

``` c++
#include <array>
#include <cassert>
#include <iostream>
#include <string>

template <typename T, auto Maxsize> class Stack {
public:
  using size_type = decltype(Maxsize);

private:
  std::array<T, Maxsize> elems;
  size_type num_elems;

public:
  Stack();
  void push(T const &elem);
  void pop();
  T const &top() const;
  bool empty() const { return num_elems == 0; }
  size_type size() const { return num_elems; }
};

template <typename T, auto Maxsize> Stack<T, Maxsize>::Stack() : num_elems(0) {}

template <typename T, auto Maxsize>
void Stack<T, Maxsize>::push(T const &elem) {
  assert(num_elems < Maxsize);
  elems[num_elems] = elem;
  ++num_elems;
}

template <typename T, auto Maxsize> void Stack<T, Maxsize>::pop() {
  assert(!elems.empty());
  --num_elems;
}

template <typename T, auto Maxsize> T const &Stack<T, Maxsize>::top() const {
  assert(!elems.empty());
  return elems[num_elems - 1];
}

/////////////////////////////////////////////////////////////

template <auto T> class Message {
public:
  void print() { std::cout << T << '\n'; }
};

extern char const s03[] = "hi"; // external linkage
char const s11[] = "hi";        // internal linkage

int main() {
  Stack<int, 20u> int_20_stack;
  int_20_stack.push(7);
  std::cout << int_20_stack.top() << '\n';
  int_20_stack.pop();

  // error: ‘double’ is not a valid type for a template non-type parameter
  // 3.3 Restrictions for Nontype Template Parameters, page 49
  // 浮点数和类类是不能做为非类型模板参数的
  // Stack<double, 22.2> double_stack;

  Stack<std::string, 40> string_stack;
  string_stack.push("hello");
  std::cout << string_stack.top() << '\n';

  ///////////////////////////////////////////////////////////

  auto size1 = int_20_stack.size();
  auto size2 = string_stack.size();
  if (!std::is_same_v<decltype(size1), decltype(size2)>) {
    std::cout << "size types differ" << '\n';
  }

  Message<42> msg1;
  msg1.print();

  static char const s[] = "hello";
  Message<s> msg2; // initialize with char const[6] "hello"
  msg2.print();

  // 3.3 Restrictions for Nontype Template Parameters, page 49
  // 这里的内容不太感冒
  Message<s03> m03;               // OK(all versions)
  Message<s11> m11;               // OK since C++11
  static const char s17[] = "hi"; // no linkage
  Message<s17> m17;               // OK since C++17
}
```

``` c++
#include <iostream>
#include <string>

template <typename T> void print(T arg) { std::cout << arg << '\n'; }

// 4.1.2 Overloading Variadic and Nonvariadic Templates, page 57
// 这里没有提供`print()`这个无参函数，但是却能正常执行，
// 这让我很诧异呀，细想一下发现确实没问题。
// 如果有两个重载的函数模板仅有一个`trailing parameter pack`的区别，
// 比如这个代码里的两个`print`模板函数，则`print(T arg)`会被优先
// 代码里的`Types... args`就是`trailing parameter pack`。
template <typename T, typename... Types>
void print(T first_arg, Types... args) {
  std::cout << "sizeof...(Types): " << sizeof...(Types) << '\n';
  std::cout << "sizeof...(args): " << sizeof...(args) << '\n';
  print(first_arg);
  print(args...);
}

// 4.1.3 Operator sizeof..., page 58
// 如果不提供`print()`无参函数，想通过`sizeof...`来计算`args`大小
// 大于`0`时才调用`print()`是不可行的，因为模板实例化时，`if`的所
// 有分支都将被实例化，所以无法通过编译
int main() {
  std::string s("world");
  print(7.5, "hello", s);
}
```

我还真不记得`->*`是啥了，一分钟读一下：“[Pointer-to-member operators: `.*` and `->*`](https://docs.microsoft.com/en-us/cpp/cpp/pointer-to-member-operators-dot-star-and-star?view=msvc-170)”这里的例子就清楚了。

``` c++
#include <iostream>

struct Node {
  int value;
  Node *left;
  Node *right;
  Node(int i = 0) : value(i), left(nullptr), right(nullptr) {}
};

auto left = &Node::left;
auto right = &Node::right;

// 4.2 Fold Expressions, page 59
// traverse tree, using fold expression:
template <typename T, typename... TP> Node *traverse(T np, TP... paths) {
  return (np->*...->*paths); // np ->* paths1 ->* paths2 ...
}

int main() {
  Node *root = new Node{0};
  root->left = new Node{1};
  root->left->right = new Node{2};

  Node *node = traverse(root, left, right);
}
```

``` c++
#include <iostream>

template <typename T> class AddSpace {
private:
  T const &ref; // refer to argument passed in constructor

public:
  AddSpace(T const &r) : ref(r) {}
  friend std::ostream &operator<<(std::ostream &os, AddSpace<T> s) {
    return os << s.ref << ' '; // output passed argument and a space
  }
};

// 4.2 Fold Expressions, page 59
// 原来这里需要用`()`把`std::cout`括起来
template <typename... Args> void print(Args... args) {
  (std::cout << ... << AddSpace(args)) << '\n';
}

int main() { print(7, 3.2, "hello"); }
```

``` c++
#include <array>
#include <complex>
#include <iostream>
#include <string>
#include <tuple>

template <typename... Args> void print(Args... args) {
  (std::cout << ... << args) << '\n';
}

// 4.4.1 Variadic Expressions, page 62
// output: 15hellohello(8,4)
template <typename... T> void print_double(T const &...args) {
  print(args + args...);
}

// 4.4.1 Variadic Expressions, page 62
// output: 8.5498
// 即：8.5 4 98
template <typename... T> void add_one(T const &...args) {
  // ERROR: 1... is a literal with too many decimal points
  // print(args + 1...);
  print(args + 1 ...);
  print((args + 1)...);
}

// 4.4.1 Variadic Expressions, page 62
template <typename T1, typename... TN>
constexpr bool is_homogeneous(T1, TN...) {
  return (std::is_same<T1, TN>::value && ...); // since C++17
}

////////////////////////////////////////////////////////////////

// 4.4.3 Variadic Class Templates, page 64
// 这里不太看得懂，这里是元编程
template <std::size_t...> struct Indices {};

template <typename T, std::size_t... Idx>
void print_by_idx(T t, Indices<Idx...>) {
  print(std::get<Idx>(t)...);
}

int main() {
  print_double(7.5, std::string("hello"), std::complex<float>(4, 2));
  add_one(7.5, 3, 'a');
  // 4.4.1 Variadic Expressions, page 62
  // 这个表达式将返回`false`
  is_homogeneous(43, -1, "hello");
  // 这个表达式将返回`true`，因为这是传值，所以`arrays become pointers`
  // 所以参数被`decay`成`const char *`
  is_homogeneous("hello", " ", "world");

  std::array<std::string, 5> arr = {"Hello", "my", "new", "!", "World"};
  print_by_idx(arr, Indices<0, 4, 3>()); // output: HelloWorld!
  auto t = std::make_tuple(12, "monkeys", 2.0);
  print_by_idx(t, Indices<0, 1, 2>()); // output: 12monkeys2
}
```

## <2022-05-30 Mon>

``` c++
#include <iostream>

class YoClass {
public:
  static int SubType;
};

// 5.1 Keyword typename, page 67
// `typename`定义一个指向类型为`T::SubType`的指针，
// 这里编译会失败，因为`YoClass`里没有一个叫`SubType`类型
// 按照书中的意思，没有这个`typename`，`T::SubType * ptr`
// 的意思将变成`YoClass::SubType`乘以`ptr`。
template <typename T> class MyClass {
public:
  void foo() { typename T::SubType *ptr; }
};

int main() {
  MyClass<YoClass> mc;
  mc.foo();
}
```

``` c++
#include <iostream>

template <typename T> void foo() {
  T x;
  std::cout << "x: " << x << '\n';
}

// 这是`foo()`的全特化，针对`bool`类型
template <> void foo<bool>() {
  bool x;
  std::cout << "x: " << std::boolalpha << x << '\n';
}

// 似乎`C++`不支持函数模板的偏特化，在整个`1. Function Templates`章
// 中找不到关于`Partial Specialization`，而`2. Class Templates`章中
// 有专门`Partial Specialization`的内容。
// 这里编译出错：
// non-class, non-variable partial specialization ‘foo<T*>’ is not allowed
// template <typename T> void foo<T *>() {
//   T *x;
//   std::cout << "x: " << static_cast<T *>(x) << '\n';
// }

// 那就用类模板的偏特化来实现吧先
template <typename T> class Foo {
public:
  void foo() {
    T x;
    std::cout << "x: " << x << '\n';
  }
};

// 2.6 Partial specialization, page 33
// 这是上面`Foo`类的偏特化版本
template <typename T> class Foo<T *> {
public:
  void foo() {
    T *x;
    std::cout << "x: " << static_cast<void *>(x) << '\n';
  }
};

int main() {
  // 5.2 Zero Initialization, page 69
  // 这里分两个部分进行测试，上半部分是`main`函数，可以看出虽然在
  // `main`函数中变量没有初始化，但是`int`型被初始化为`0`，`bool`
  // 型初始化为`false`，指针型初始化为`nullptr`，而下半部分函数中
  // 的未初始化变量则不然。
  int i, j{};
  bool t, f{};
  int *p1, *p2{};
  std::cout << "i: " << i << '\n';
  std::cout << "j: " << j << '\n';
  std::cout << "t: " << std::boolalpha << t << '\n';
  std::cout << "f: " << std::boolalpha << f << '\n';
  std::cout << "p1: " << static_cast<void *>(p1) << '\n';
  std::cout << "p2: " << static_cast<void *>(p2) << '\n';

  std::cout << "--------------------\n";

  foo<int>();
  foo<bool>();
  Foo<int *>().foo();
}
```

上面代码中提到函数模板偏特化没有实现，可能`C++`标准不支持，可以使用`tag`来实现，我的理解它就像`boost.gil`中的`jpeg_tag()`一样：

``` c++
// 尝试：函数重载实现函数偏特化
// 这里使用了`tag`就像是`boost.gil`中的`jpeg_tag()`一样
// 这里与前段代码中的`foo()`函数有所不同，必须加上形参of

#include <iostream>

struct TypeTag {};
struct TypePtrTag {};

template <typename T> struct TagDispatchTrait { using Tag = TypeTag; };

template <typename T> struct TagDispatchTrait<T *> { using Tag = TypePtrTag; };

template <typename T> void foo(T a, TypeTag) {
  T x;
  std::cout << "TypeTag    x: " << x << '\n';
}

template <typename T> void foo(T a, TypePtrTag) {
  T *x;
  std::cout << "TypePtrTag x: " << static_cast<void *>(x) << '\n';
}

// 这里与前段代码中的`foo()`函数有所不同，必须加上形参，输出如下：
// TypeTag    x: 0
// TypePtrTag x: 0
// 居然结果也都为`0`，但是目前来说结果不重要。
template <typename T> void foo(T a) {
  return foo(a, typename TagDispatchTrait<T>::Tag{});
}

int main() {
  int i, *p;
  foo(i);
  foo(p);
}
```

## <2022-05-31 Tue>

``` c++
// 5.4 Templates for Raw Arrays and String Literals, page 72

#include <iostream>

template <typename T> struct MyClass; // primary template

template <typename T, std::size_t SZ>
struct MyClass<T[SZ]> // partial specialization for arrays of known bounds
{
  static void print() { std::cout << "print() for T[" << SZ << "]\n"; }
};

template <typename T, std::size_t SZ>
struct MyClass<T (&)[SZ]> // partial spec. for references to arrays of known
                          // bounds
{
  static void print() { std::cout << "print() for T(&)[" << SZ << "]\n"; }
};

template <typename T>
struct MyClass<T[]> // partial specialization for arrays of unknown bounds
{
  static void print() { std::cout << "print() for T[]\n"; }
};

template <typename T>
struct MyClass<T (&)[]> // partial spec. for references to arrays of unknown
                        // bounds
{
  static void print() { std::cout << "print() for T(&)[]\n"; }
};

template <typename T>
struct MyClass<T *> // partial specialization for pointers
{
  static void print() { std::cout << "print() for T *\n"; }
};

template <typename T1, typename T2, typename T3>
void foo(int a1[7], int a2[], // pointers by language rules
         int (&a3)[42],       // reference array of known bound
         int (&x0)[],         // reference to array of unknown bound
         T1 x1,               // passing by value decays
         T2 &x2, T3 &&x3)     // passing by reference
{
  MyClass<decltype(a1)>::print(); // uses MyClass<T *>
  MyClass<decltype(a2)>::print(); // uses MyClass<T *>
  MyClass<decltype(a3)>::print(); // uses MyClass<T(&)[SZ]>
  MyClass<decltype(x0)>::print(); // uses MyClass<T(&)[]>
  MyClass<decltype(x1)>::print(); // uses MyClass<T *>
  MyClass<decltype(x2)>::print(); // uses MyClass<T(&)[]>
  MyClass<decltype(x3)>::print(); // uses MyClass<T(&)[]>
}

int main() {
  int a[42];
  MyClass<decltype(a)>::print(); // uses MyClass<T[SZ]>

  extern int x[];                // forward declare array
  MyClass<decltype(x)>::print(); // uses MyClass<T[]>

  foo(a, a, a, x, x, x, x);
}

int x[] = {0, 8, 15}; // define forward-declared array
```

``` c++
#include <cassert>
#include <deque>
#include <iostream>

template <typename T> class Stack {
private:
  std::deque<T> elems;

public:
  void push(T const &elem) { elems.push_back(elem); }
  void pop() {
    assert(!elems.empty());
    elems.pop_back();
  }
  T const &top() const {
    assert(!elems.empty());
    return elems.back();
  }
  bool empty() const { return elems.empty(); }

  template <typename T2> Stack &operator=(Stack<T2> const &op2);

  // 5.5 Member Templates, page 75
  // 书上说为了能访问`op2`的所有成员，你可以声明所有其它的`stack`是友元
  // 但是我为什么发现这段代码没有用处呢？
  // 因为对于这里有两个版本的赋值操作符，第一个版本声明了一个`tmp`临时
  // 变量，这当然不用声明友元，第二个版本没有这个临时变量而是直接操作的
  // `op2`变量，所以它需要下面个声明。
  // 这里的`T`省略了，因为没有用到，所以可以省略
  template <typename> friend class Stack;
};

// 5.5 Member Templates, page 75
// 第一个版本的赋值操作符
// template <typename T>
// template <typename T2>
// Stack<T> &Stack<T>::operator=(Stack<T2> const &op2) {
//   Stack<T2> tmp(op2);
//
//   elems.clear();
//   while (!tmp.empty()) {
//     elems.push_front(tmp.top());
//     tmp.pop();
//   }
//   return *this;
// }

// 5.5 Member Templates, page 76
// 第二个版本的赋值操作符
template <typename T>
template <typename T2>
Stack<T> &Stack<T>::operator=(Stack<T2> const &op2) {
  elems.clear();
  elems.insert(elems.begin(), op2.elems.begin(), op2.elems.end());
  return *this;
}

// output:
// int_stack1.top(): 6
// int_stack1.top(): 5
// int_stack1.top(): 4
// float_stack.top(): 6
// float_stack.top(): 5
// float_stack.top(): 4

int main() {
  Stack<int> int_stack1, int_stack2;
  Stack<float> float_stack;

  int_stack1.push(1);
  int_stack1.push(2);
  int_stack1.push(3);
  int_stack2.push(4);
  int_stack2.push(5);
  int_stack2.push(6);
  float_stack.push(1.0);
  float_stack.push(2.0);
  float_stack.push(3.0);

  int_stack1 = int_stack2;
  while (!int_stack1.empty()) {
    std::cout << "int_stack1.top(): " << int_stack1.top() << '\n';
    int_stack1.pop();
  }

  float_stack = int_stack2;
  while (!float_stack.empty()) {
    std::cout << "float_stack.top(): " << float_stack.top() << '\n';
    float_stack.pop();
  }
}
```
