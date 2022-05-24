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
// 因为模板推导不了返回值`RT`的类型，所以调用的时候：
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
// 这里有一个大缺隐就是返回值可能会是引用类型，因`T`可能是一个引用
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
