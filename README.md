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

// output:
// T const &(T const &, T const &, T const &)
// T const &(T const &, T const &)
// T const &(T const &, T const &)
// 68
// -------------
// T const &(T const &, T const &, T const &)
// char const *(char const *, char const *)
// char const *(char const *, char const *)
// Segmentation fault (core dumped)

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
  // 浮点数类型和类类型是不能做为非类型模板参数的
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
// 这里与前段代码中的`foo()`函数有所不同，必须加上形参

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
  // 5.5 Member Templates, page 75
  // 这里为什么用`std::deque`也是有讲究的，因为它提供了`push_front()`这
  // 样的函数，我的理解是要不然赋值后会变成倒序
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

## <2022-06-01 Wed>

``` c++
#include <iostream>
#include <string>

class BoolString {
private:
  std::string value;

public:
  BoolString(std::string const &s) : value(s) {}

  template <typename T = std::string> T get() const {
    return value; // get value (converted to T)
  }
};

// Specialization of Member Function Template, page 78
// full specialization for BoolString::get<>() for bool
// 注意，你不需要声明特化函数，同时`C++`也声明不了特化函数，你只
// 需要定义它们即可。
// 因为这里是一个全特化函数，如果在头文件里（我这里放在源文件里）
// 用`inline`关键字避免出现错误（如果在别的翻译单元里已经包含了
// 这个定义的话）
// 书上还说成员函数模板支持偏特化和全特化，这里是全特化，偏特化
// 是什么样子的？
template <> inline bool BoolString::get<bool>() const {
  return value == "true" || value == "1" || value == "on";
}

// Special Member Function Templates, page 79
// 这里的`special member function`指的是什么函数？然后紧接着又说
// `Member template`不能算做`special member function`因为它们复
// 制和移动对象。
// `template constructors`，`template assignment operators`不会替换
// `predefined constructors or assignment operators`。
// 1，`template constructor or assignment operator`比`predefined copy/move
// constructor or assignment operator`更好的匹配。
// 2，对复制移动构造函数模板化是不太容易的。

// 5.5.1 The .template Construct, page 79
// 这里暂时没看懂

int main() {
  std::cout << std::boolalpha;
  BoolString s1("hello");
  std::cout << s1.get() << '\n';
  std::cout << s1.get<bool>() << '\n';
  BoolString s2("on");
  std::cout << s2.get<bool>() << '\n';
}
```

``` c++
#include <array>
#include <iostream>

// 5.6 Variable Templates, page 81
// 这个我翻译成变量模板，对我来说它比较新，代码看起来不太直观。
template <typename T> constexpr T pi{3.1415926535897932385};

template <int N>
std::array<int, N> arr{}; // array with N elements, zero-initialized

template <auto N>
constexpr decltype(N) dval = N; // type of dval depends on passed value

// Variable Templates for Data Members, page 82
template <typename T> class my_numeric_limits {
public:
  static constexpr bool is_signed = false;
};

// Variable Templates for Data Members, page 82
// 有了这个定义，就可以写出`main`函数里那样的代码了
template <typename T> constexpr bool issigned = my_numeric_limits<T>::is_signed;

// Type Traits Suffix _v, page 83
// 标准库里那些`_v`，`_t`的伎俩就是变量模板的作用
template <typename T> constexpr bool is_const_v = std::is_const<T>::value;

int main() {
  std::cout << pi<double> << '\n';
  std::cout << pi<float> << '\n';

  std::cout << dval<'c'> << '\n'; // N has value 'c' of type char
  arr<10>[0] = 42;
  for (std::size_t i = 0; i < arr<10>.size(); ++i) {
    std::cout << arr<10>[i] << ' ';
  }
  std::cout << '\n';

  // 这里输出都是相同的，只是前者使用了变量模板而已
  std::cout << std::boolalpha << issigned<char> << '\n';
  std::cout << std::boolalpha << my_numeric_limits<char>::is_signed << '\n';
}
```

``` c++
#include <deque>
#include <iostream>
#include <vector>

// 5.7 Template Template Parameters, page 83
// 直观的理解这里就是让这种写法；
// Stack<int, std::vector<int>> vstack;
// 变成这种写法：
// Stack<int, std::vector> vstack;

template <typename T,
          template <typename Elem, typename Alloc = std::allocator<Elem>>
          class Cont = std::deque>
class StackForPage85 {};

// 这里的`Elem`可以省略，因为它没有被使用，成员函数也要进行相应修改
// template <typename T,
//           template <typename Elem> class Cont = std::deque>
template <typename T, template <typename> class Cont = std::deque> class Stack {
private:
  Cont<T> elems;

public:
  void push(T const &);
  void pop();
  T const &top() const;
  bool empty() const { return elems.empty(); }
};

template <typename T, template <typename> class Cont>
void Stack<T, Cont>::push(T const &elem) {
  elems.push_back(elem);
}

int main() {
  // Template Template Argument Matching, page 85
  // 这里确实正如书上所说，如果使用`-std=c++11`或`-std=c++14`都会编译失败
  // Stack<int, std::vector> vstack;
  // 通过改成`StackForPage85`的声明就可以编译通过了，因为在`c++17`之前的
  // 标准库里`std::deque`不止有一个参数，所以这里匹配不成功
  StackForPage85<int, std::vector> v;
}
```

``` c++
#include <iostream>
#include <utility>

class X {};

void g(X &) { std::cout << "g() for variable\n"; }

void g(X const &) { std::cout << "g() for constant\n"; }

void g(X &&) { std::cout << "g() for movable object\n"; }

// 6.1 Perfect Forwarding, page 93
// 为什么会有完美转发？试想下面这三个`f()`重载函数要用模板代替的话
// 显然`template <typename T> void f(T val);`只能匹配前两个`f()`
// void f(X &val) { g(val); }

// void f(X const &val) { g(val); }

// void f(X &&val) {
//   g(std::move(val)); // need std::move() to call g(X &&)
// }

// 如果上面三个`f()`重载函数不注释掉的话，则不会调用此模板函数
// output:
// f<T>(T &&) called: g() for variable
// f<T>(T &&) called: g() for constant
// f<T>(T &&) called: g() for movable object
// f<T>(T &&) called: g() for movable object
// 6.1 Perfect Forwarding, page 93
// 我觉得这段比较重要，介绍了`T&&`和`X&&`的不同，具体可看原文。
// `X&&`对于`X`来说是一个特殊类型，表示右值引用，它仅能被绑定到
// 一个可移动对象上（`prvalue`直观理解为临时对象，`xvalue`直观
// 理解为`std::move()`传递的对象），它总是`mutable`；而`T&&`对于
// `T`来说是`T`声明了一个`forwarding reference`（也叫`universal
// reference`），它可以绑定`mutable`，`immutable`等等。
// 注意：`T::iterator&&`是`rvalue reference`而不是`forwarding reference`
template <typename T> void f(T &&val) {
  std::cout << "f<T>(T &&) called: ";
  g(std::forward<T>(val)); // perfect forward val to g()
}

int main() {
  X v;
  X const c;

  f(v);
  f(c);
  f(X());
  // 6.1 Perfect Forwarding, page 93
  // 书中说：move semantics is not passed through，
  // 移动语义是不会传递的，没理解这句话！
  // 这里为什么要加上`std::move`，因为虽然第三个`f(X &&)`参数是右值引用
  // 但是如果传参的是表达式的话，它的行为就是左值（nonconstant lvalue）
  f(std::move(v));
}
```

``` c++
#include <iostream>
#include <string>
#include <utility>

class Person {
private:
  std::string name;

public:
  template <typename STR>
  explicit Person(STR &&n) : name(std::forward<STR>(n)) {
    std::cout << "TMPL-CONSTR for '" << name << "'\n";
  }

  Person(Person const &p) : name(p.name) {
    std::cout << "COPY-CONSTR for '" << name << "'\n";
  }

  Person(Person &&p) : name(std::move(p.name)) {
    std::cout << "MOVE-CONSTR for '" << name << "'\n";
  }
};

int main() {
  std::string s = "sname";
  Person p1(s);
  Person p2("tmp");
  Person const p2c("ctmp");
  Person p3c(p2c);
  // 6.2 Special Member Function Templates, page 97
  // 上面自己曾问`special member function`是啥？这里再次出现
  // 这里为什么会出错目前还不理解，端午节后再战
  // Person p3(p1);
  Person p4(std::move(p1));
}
```

## <2022-06-07 Tue>

``` c++
#include <iostream>
#include <string>
#include <utility>

// 6.4 Using enable_if<>, page 100
// `main()`函数中的那个`Person p3(p1)`的错误可以用`std::enable_if`来
// 解决
template <typename T>
// using EnableIfString =
//     std::enable_if_t<std::is_convertible_v<T, std::string>>;
// 6.4 Using enable_if<>, page 101
// 上行`std::is_convertible`它要求类型是隐式可转换的，通过使用
// `std::is_constructible<>`，我们还允许显式转换用于初始化
// 注意它们的参数位置是颠倒的
using EnableIfString =
    std::enable_if_t<std::is_constructible_v<std::string, T>>;

class Person {
private:
  std::string name;

public:
  // 6.4 Using enable_if<>, page 100
  // 因此这里要改成：
  // template <typename STR>
  template <typename STR, typename = EnableIfString<STR>>
  explicit Person(STR &&n) : name(std::forward<STR>(n)) {
    std::cout << "TMPL-CONSTR for '" << name << "'\n";
  }

  Person(Person const &p) : name(p.name) {
    std::cout << "COPY-CONSTR for '" << name << "'\n";
  }

  Person(Person &&p) : name(std::move(p.name)) {
    std::cout << "MOVE-CONSTR for '" << name << "'\n";
  }
};

int main() {
  std::string s = "sname";
  Person p1(s);
  Person p2("tmp");
  Person const p2c("ctmp");
  Person p3c(p2c);
  // 6.2 Special Member Function Templates, page 97
  // 上面自己曾问`special member function`是啥？这里再次出现这里为什
  // 么会出错目前还不理解，端午节后再战这里为什么会出错，书中说根据
  // `C++`的重载解析规则，在此代码中：
  // template <typename STR>
  // Person(STR &&n)
  // 比：
  // Person(Person const &p)
  // 更好的匹配，所以会出错，因为你不能把`Person`赋给`std::string`的
  // `name`成员，书上同时也说了，为什么前者比后者更好的匹配，因为对
  // 于前者来说，`STR`只是被替换为`Person &`，而后者必须要将它转化为
  // `const`类型。同时书上也说了，可以提供一个`Person(Person &p)`这
  // 样的非`const`的拷贝构造函数，但是考虑到派生类对象，成员模板依然
  // 是更好的匹配，因此当传传递的参数是`Person`或可以转化为`Person`的
  // 表达式时你真正需要做的是禁用成员模板。
  // 然后引出将要讲的内容：std::enable_if<>
  Person p3(p1);
  Person p4(std::move(p1));
}
```

``` c++
#include <iostream>

class C {
public:
  C() = default;
  // 将预定义的拷贝构造函数显示删除不能达到目的，编译不通过
  // C(C const &) = delete;

  // Disable Special Member Functions, page 102
  // 可以用这个`tricky solution`：
  // user-define the predefined copy constructor as deleted
  // (with convertion to volatile to enable better matches)
  // 经过这样的修改就可以输出`tmpl copy ...`了，这里的`volatile`
  // 用得很精髓啊！完全看不懂。
  C(C const volatile &) = delete;
  template <typename T> C(T const &) { std::cout << "tmpl copy constructor\n"; }
};

int main() {
  C x;

  // Disable Special Member Functions, page 102
  // 如何让这里能调用到成员模板呢？即可以输出上面的`tmpl copy ...`
  C y{x};
}
```

``` c++
#include <iostream>
#include <string>
#include <type_traits>

// 7.2.1 Passing by Constant Reference, page 109
// 这里：“What is the callee doing with that address?”后面的内容不理解，
// 跟上下文的关系是什么？直接理解的话是因为传引用实际上就是传指针，被
// 调用者，即`callee`可能会对该地址做任何事情，这样的话函数返回后调用者
// 需要重新加载这些值的操作可能会比较昂贵。
// 原文：“You may be thinking that we are passing by constant reference:
// Cannot the compiler deduce from that that no change can happen? ...”
// 谷歌的翻译：“编译器不能从中推断出不会发生任何变化吗？不幸的是，情况
// 并非如此，因为调用者可能通过它自己的非`const`引用来修改被引用的对象”
// 这里好像与：
// Passing by Reference Does Not Decay, page 109
// 联系起来了，因为`call parameter`被声明为`T const &`，`template parameter T`
// 本身不会被推导为`const`，比如：
template <typename T> void print_ref(T const &arg) {}

// 7.2.2 Passing by Nonconstant Reference, page 110
template <typename T> void out_ref(T &arg) {
  if (std::is_array<T>::value) {
    std::cout << "got array of " << std::extent<T>::value << " elems\n";
  }
}

// 7.2.2 Passing by Nonconstant Reference, page 111
template <typename T> void out_ref1(T &arg) {
  static_assert(!std::is_const<T>::value,
                "out parameter of foo<T>(T &) is const");
}

template <typename T, typename = std::enable_if_t<!std::is_const<T>::value>>
void out_ref2(T &arg) {}

// 这里编译报错（-std=c++20）：
// main.cpp:37:10: error: expression must be enclosed in parentheses
//    42 | requires !std::is_const_v<T> void out_ref3(T & arg) {}
//       |          ^~~~~~~~~~~~~~~~~~~
//       |          (                  )
// template <typename T>
// requires !std::is_const_v<T> void out_ref3(T & arg) {}

// 7.2.3 Passing by Forwarding Reference, page 112
// 这里的问题是，如果`T`被推导成比如是`int &`类型，则`x`是需要初始化的
template <typename T>
void pass_ref(T &&arg) { // arg is a `forwarding reference`
  T x;
}

int main() {
  std::string const c = "hi";
  print_ref(c);    // T deduced as `std::string`, arg is `std::string const &`
  print_ref("hi"); // T deduced as `char[3]`, arg is `char const(&)[3]`
                   // Passing by Reference Does Not Decay, page 110
                   // 这样在`print_ref`中的局部变量`T`不是`const`，这里好像和
                   // 109页的内容对应起来了？

  // 7.2.2 Passing by Nonconstant Reference, page 110
  std::string s = "hi";
  out_ref(s); // OK: T deduced as `std::string`, arg is `std::string &`
  /*
  std::string return_string();
  out_ref(std::string("hi")); // ERROR: not allowed to pass a temporary(prvalue)
  out_ref(return_string());   // ERROR: not allowed to pass a temporary(prvalue)
  out_ref(std::move(s));      // ERROR: not allowed to pass an xvalue
  */
  int arr[4];
  out_ref(arr); // OK: T deduced as `int[4]`, arg is `int(&)[4]`

  // 上面那三个错误是可以理解的，如果传入`const`引用，则上面三个表达式却编译通过
  // 这样的话在`out_ref`中传入的参数就不能被修改了，它跟`print_ref`正好相反了
  // 正如110页所说，模板采用了一个小伎俩才能让这种情况发生，具体啥伎俩书上目前没
  // 讲，但是如果你想禁用这种功能，即把`const`对象传给非`const`引用这个功能，你
  // 可以试试另三种`out_ref`函数。
  out_ref(std::move(c));
  out_ref("hi");

  // error: static assertion failed: out parameter of foo<T>(T &) is const
  // out_ref1(std::move(c));
  // out_ref1("hi");

  // out_ref2(std::move(c));
  // out_ref2("hi");

  // out_ref3(std::move(c));
  // out_ref3("hi");

  // 7.2.3 Passing by Forwarding Reference, page 112
  pass_ref(42); // OK: T deduced as int
  int i;
  // pass_ref(i); // ERROR: T deduced as `int &`
}
```

``` c++
#include <functional> // for std::cref()
#include <iostream>
#include <string>

void print_string(std::string const &s) { std::cout << s << '\n'; }

// 7.3 Using std::ref() and std::cref(), page 113
// 这里讲了`std::ref`和`std::cref`要有一种能力，一个隐式转化回原类型的能力，
// 或者生成原始对象的能力。
// 虽然是传值这里，但是有了`std::ref`和`std::cref`整得就像传引用一样，它们
// 的内部实现是：创建了一个`std::reference_wrapper<>`对象，该对象引用了原
// 始参数并以值传递该对象。
// 同时书中下面的内容讲到了为什么这里还需要一个`print_string`函数呢？
// 因为上面讲的`std::ref`和`std::cref`的转化为原类型的能力就体现在这里。
template <typename T> void print_t(T arg) {
  print_string(arg); // might convert arg back to std::string
}

// 7.3 Using std::ref() and std::cref(), page 113
// 下面的`print_v(std::cref(s));`会报错，因为`std::reference_wrapper<>`没有
// `<<`的定义
template <typename T> void print_v(T arg) { std::cout << arg << '\n'; }

int main() {
  std::string s = "hello";
  print_t(s);            // print s passed by value
  print_t(std::cref(s)); // print s passed "as if by reference"

  // print_v(std::cref(s));
}
```

``` c++
#include <iostream>
#include <type_traits>

// 7.5 Dealing with Return Values, page 117
// 就像在前面讲的，模板参数`T`不能保证它不是一个引用，`T`在某些情况下
// 可能会被隐式的推导成一个引用类型。
template <typename T> T ret_ref(T &&p) { // p is a forwarding reference
  return T{}; // OOPS: returns by reference when called for lvalues
}

template <typename T> T ret_val(T p) { // Note: T might become a reference
  return T{}; // OOPS: returns a reference if T is a reference
}

// 7.5 Dealing with Return Values, page 118
template <typename T> typename std::remove_reference<T>::type ret_val1(T p) {
  return T{}; // always returns by value
}

template <typename T>
auto ret_val2(T p) { // by-value return type deduced by compiler
  return T{};        // always returns by value
}

int main() {
  int x;
  // ret_ref(x);        // ERROR
  // ret_val<int &>(x); // ERROR

  // 7.5 Dealing with Return Values, page 118
  // 按照书上的说法，下面的两个应该能编译成功的，但是为什么会出错：
  // error: cannot bind non-const lvalue reference of type ‘int&’ to an rvalue
  // of type ‘int’
  ret_val1<int &>(x); // ERROR
  ret_val2<int &>(x); // ERROR
}

// 7.6 Recommended Template Parameter Declaration, page 118
// 这里很值得一读
// General Recommendations, page 119
// 要记住这里的选项，我只记录第一个，其它看原文：默认情况下，声明模板参数为
// 传值类型，因为它简单并且可以使用字面字符串做为参数，对于小的参数，临时对象
// 或者可移动对象，性能方面还可以，调用者可以使用`std::ref()`和`std::cref()`
// 避免为大对象的传递付出昂贵的拷贝代价。
// The std::make_pair() Example, page 120
// 这里讲了从`C++98`，`C++03`至`C++11`，`std::make_pair()`这个函数的演变过程
// 值得一看。
```

## <2022-06-08 Wed>

``` c++
#include <iostream>

template <unsigned p, unsigned d> struct do_is_prime {
  static constexpr bool value = (p % d != 0) && do_is_prime<p, d - 1>::value;
};

// 8.1 Template Metaprogramming, page 125
// 1，递归展开`do_is_prime<>`去迭代所有除数，从`p/2`至`2`
// 2，编特化版本，即此结构体，`d`为`2`时做为结束递归的条件
template <unsigned p> struct do_is_prime<p, 2> {
  static constexpr bool value = (p % 2 != 0);
};

template <unsigned p> struct is_prime {
  static constexpr bool value = do_is_prime<p, p / 2>::value;
};

// special cases (to avoid endless recursion with template instantiation)
template <> struct is_prime<0> { static constexpr bool value = false; };
template <> struct is_prime<1> { static constexpr bool value = false; };
template <> struct is_prime<2> { static constexpr bool value = true; };
template <> struct is_prime<3> { static constexpr bool value = true; };

////////////////////////////////////////////////////////////////////

// 8.2 Computing with constexpr, page 126
// 这是`c++11`的写法，`constexpr`限制只能使用一条语句，所以使用三元运算符
constexpr bool do_is_prime_c11(unsigned p, unsigned d) {
  return d != 2
             ? (p % d != 0) &&
                   do_is_prime_c11(p, d - 1) // check this and smaller divisors
             : (p % 2 != 0);                 // end recursion if devisor is 2
}

constexpr bool is_prime_c11(unsigned p) {
  return p < 4 ? !(p < 2) // handle special cases
               : do_is_prime_c11(
                     p, p / 2); // start recursion with divisor with p/2
}

////////////////////////////////////////////////////////////////////

// 8.2 Computing with constexpr, page 126
// 这是`c++14`的写法，更直观，像平常写的代码
constexpr bool is_prime_c14(unsigned int p) {
  for (unsigned int d = 2; d < p / 2; ++d) {
    if (p % d == 0) {
      return false;
    }
  }
  return p > 1;
}

int main() {
  // 为了打印输出，要加上`::value`
  std::cout << "9 is prime: " << std::boolalpha << is_prime<9>::value << '\n';

  // 8.2 Computing with constexpr, page 126
  // 在某些上下文中需要一个编译期的值（比如数组长度或者非类型模板参数），编译器
  // 在编译期间会试图去执行`constexpr`的调用，如果不成功则生成一个错误（比如一
  // 个需要的常量在最后才能生成）
  // 在某些其它上下文中，编译器可能在编译时尝试求值，也可能不尝试，但是如果这样
  // 的计算失败，则不会发出错误，而是将调用保留为运行时调用。
  // 上面是对原文的翻译，即`constexpr`的求值是在编译期还是运行期，现在不得而知
  std::cout << "9 is prime: " << std::boolalpha << is_prime_c11(9) << '\n';
  std::cout << "9 is prime: " << std::boolalpha << is_prime_c14(9) << '\n';
}

// 8.3 Execution Path Selection with Partial Specialization, page 128
// 因为函数模板不支持偏特化，所以有以下几种选择：
// 1, Use classes with static functions,
// 2, Use `std::enable_if`, introduced in Section 6.3 on page 98,
// 3, Use the SFINAE feature, which is introduced next, or
// 4, Use the compile-time `if` feature, available since C++17, which is
//    introduced below in Section 8.5 on page 135.
// 早就听说`SFINAE`了，但是不理解它能做什么？现在知道它的一个功能了。
```

``` c++
#include <iostream>
#include <vector>

// number of elements in a raw array
template <typename T, unsigned N> std::size_t len(T (&)[N]) {
  std::cout << "std::size_t len(T(&)[N]): ";
  return N;
}

// number of elements for a type having size_type
template <typename T> typename T::size_type len(T const &t) {
  std::cout << "T::size_type len(T const &): ";
  return t.size();
}

// 8.4 SFINAE (Substitution Failure Is Not An Error), page 130
// 当替换一个候选对象的返回类型没有意义时，忽略它会导致编译器选择另一个
// 参数匹配更差的候选对象。
// 这句话应该说的就是这个`fallback`函数
// fallback for all other types
// 页解备注，page 131
// 这样一个`fallback`函数常常用来提供一个更有用的默认操作，抛出异常或者
// 包括一个断言并显示一个有用的错误信息。
std::size_t len(...) {
  std::cout << "std::size_t len(...): ";
  return 0;
}

// SFINAE and Overload Resolution, page 132
// 这里讲的“we SFINAE out a function"，意思是忽略掉这个函数，对于
// 标准库中“shall not participate in overload resolution unless...”
// 就是指在某种情况下忽略掉某个函数模板。
// 这里也提到了95页的内容，再次提出某些情况下，函数类的成员函数模板
// 可能比预定义的复制和移动构造函数更好的匹配，在这种情况下就需要用
// 到`SFINAE`来忽略函数模板，使用预定义的构造函数，就像在`std::thread`
// 中的那样。

// 8.4.1 Expression SFINAE with decltype, page 133
// 注：将`main()`函数中的`len(x)`放开，依然报没有`size()`的错误。
// 书中所说下面的这个函数是想当`T`有一个`size_type()`成员但是没有
// `size()`成员时被忽略。在函数声明中没有任何对`size()`成员的要求，
// 当函数模板被选中时将产生一个错误。
// 这里为什么要求进行`(void)`的强转？书上说是为了防止用户定义的逗号
// 操作符重载的影响。
template <typename T>
auto len(T const &t) -> decltype((void)(t.size()), T::size_type()) {
  return t.size();
}

int main() {
  int a[10];
  std::cout << len(a) << '\n';
  std::cout << len("tmp") << '\n';

  std::vector<int> v;
  std::cout << len(v) << '\n';

  int *p;
  // 8.4 SFINAE (Substitution Failure Is Not An Error), page 131
  // 如果没有上面的`fallback`版本的`len()`函数，则无法编译成功
  std::cout << len(p) << '\n';

  std::allocator<int> x;
  // 这里编译错误是因为`std::allocator`没有`size()`成员函数
  // std::cout << len(x) << '\n';
}
```

``` c++
#include <iostream>
#include <string>

// 4.1.3 Operator sizeof..., page 58
// 如果不提供`print()`无参函数，想通过`sizeof...`来计算`args`大小
// 大于`0`时才调用`print()`是不可行的，因为模板实例化时，`if`的所
// 有分支都将被实例化，所以无法通过编译
// template <typename T, typename... Types>
// void print(T first_arg, Types... args) {
//   std::cout << first_arg << '\n';
//   if (sizeof...(args) > 0) {
//     print(args...);
//   }
// }

// 8.5 Compile-Time if, page 134
// 这里提到，前现讲的偏特化，`SFINAE`和`std::enable_if`是允许我们可以启用或禁
// 用整个模板，这里的`compile-time if`是允许我们启用或禁用特定的语句。书中提
// 到的之前第4章的内容我还有印象，即上面那个被注释掉的函数，可以使用下面方法
// 来实现。
template <typename T, typename... Types>
void print(T const &first_arg, Types const &...args) {
  std::cout << first_arg << '\n';
  if constexpr (sizeof...(args) > 0) {
    print(args...); // code only valid if `sizeof...(args) > 0` (since C++17)
  }
}

// 8.5 Compile-Time if, page 134
// 这里提到了`Section 1.1.3`，我翻回去看了下增加记忆。
// 说模板实例化分为两个阶段：
// 1，`definition time`阶段，没有进行实例化
//  - 检查语法错误，比如有没有缺少分号
//  - 使用不依赖于模板参数的未知名称（类型名，函数名）会被发现
//  - 检查不依赖于模板参数的静态断言
// 2，`instantiation time`阶段，模板代码再次被检查以确保所有代码有效
// 这里我的理解应该是对的吧？
// `else`分支是可能会被丢弃的如果`if constexpr`表达式成功的话
// 书中注意`(even if discarded)`我的理解是，即这个`else`分支被丢弃，
// 编译器同样报错，事实也确实如此。
template <typename T> void foo(T t) {
  if constexpr (std::is_integral_v<T>) {
    if (t > 0) {
      foo(t - 1); // OK
    }
  } else {
    undeclared(
        t); // error if not declared and not discarded (i.e. T is not integral)
    // 下面这个始终编译出错
    undeclared(); // error if not declared (even if discarded)
    // 下面这个始终编译出错
    static_assert(false, "no integral"); // always asserts (even if discarded)
    static_assert(!std::is_integral_v<T>, "no integral"); // OK
  }
}

int main() {
  std::string s("world");
  print(7.5, "hello", s);

  foo<int>(0);
}
```

``` c++
#include <iostream>
#include <limits>

void foo(int) { std::cout << "foo()\n"; }

// 8.5 Compile-Time if, page 135
// `if constexpr`可以用在任何函数中，不仅仅存在于模板中，比如：
int main() {
  if constexpr (std::numeric_limits<char>::is_signed) {
    foo(42); // OK
  } else {
    // 这里`if constexpr`用在`main()`函数中的表现和用在模板上不一样
    // 在上面代码中，用在模板函数中，`else`实例化时被丢弃，这里好像
    // 没有，编译始终出错，下面这三个语句都不能编译通过。
    undeclared(42);                   // error if undeclared() not declared
    static_assert(false, "unsigned"); // always asserts (even if discarded)
    static_assert(!std::numeric_limits<char>::is_signed,
                  "char is unsigned"); // OK
  }
}
```
