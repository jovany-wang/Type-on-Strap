---
layout: post
title:  "C++模版函数的偏特化"
date:   2020-09-18
---

<p class="intro">C++标准并不支持函数模板偏特化，然而在实际开发中，我们确实需要对一些函数模板进行偏特化。本文将介绍几种函数模板偏特化的通用方案。</p>

## 1. 什么是偏特化
### 1.1 类模板偏特化
偏特化是相对于全特化而言的，即只特化了部分模板参数，如下：
```c++
// 类模板偏特化demo
template <typename T, typename Allocator_T>
class MyVector {
public:
    MyVector() {
        std::cout << "Normal version." << std::endl;
    }
};

template <typename T>
class MyVector<T, DefaultAllocator> {
public:
    MyVector() {
        std::cout << "Partial version." << std::endl;
    }
};

MyVector<int, MyAnotherAllocator> v1;
MyVector<int, DefaultAllocator> v2;
```
输出结果:
```shell
Normal version.
Normal version.
Partial version.
```
后面的一个MyVector是一个偏特化版本，其只特化了`Allocator_T`这一个模板参数为`DefaultAllocator`。通过输出结果也可以看出来，其中v1， v2使用上面的一个类定义，而v3使用的是下面的特化版的类。
### 1.2 函数模板偏特化
和类模板偏特化同样的道理，我们尝试去对一个函数进行偏特化：
```c++
/// 函数模板偏特化demo
template <typename A, typename B>
void f(A a, B b) {
    std::cout << "Normal version." << std::endl;
}

template <typename A>
void f<A, int>(A a, int b) {
    std::cout << "Partial version." << std::endl;
}

// 测试代码
int a = 10;
double b = 12;
f(a, b);
f(a, a);
```
这段代码的意图很简单，就是期望在调`f()`的时候，如果第二个参数是int,就走到下面一个偏特化版本的`f()`里。然后这段代码编译会出现如下错误：
```shell
func_partial_demo.cc:9:6: error: function template partial specialization is not allowed
void f<A, int>(A a, int b) {
     ^~~~~~~~~
1 error generated.
```
编译器给出的错误信息也很明显，就是说我不支持模板函数的偏特化。

## 2. 实现方案
但事实上前面也提到，这种函数模板偏特化的需求其实在实际开发中非常常见，因此我们需要使用一些技巧达到对函数模板进行偏特化的目的。

### 2.1 借助类模板偏特化
由于类可以进行偏特化处理，因此一种非常直观的方案就是使用Functor代替函数，并实现`operator()`。
```
// 借助类模板偏特化demo
template <typename A, typename B>
class F {
public:
    F(A a, B b): a_(a), b_(b) {}

    void operator() () {
        // 使用a_, b_作为函数的参数
        std::cout << "Normal version." << std::endl;
    }
private:
    A a_;
    B b_;
};

template <typename A>
class F<A, int> {
public:
    F(A a, int b): a_(a), b_(b) {}

    void operator() () {
        // 使用a_, b_作为函数的参数
        std::cout << "Partial version." << std::endl;
    }
private:
    A a_;
    int b_;
};
// 测试代码
int a = 10;
double b = 12;

F<int, double>(a, b)();   // 输出 Normal version.
F<int, int>(a, a)();      // 输出 Partial version.
```
当然这里你不去实现`operator()`方法其实问题也不大，你可以继续使用`f`作为方法名，然后调用的时候调用该对象的f方法即可。

### 2.2 使用标签分发
C++标准虽然不支持函数模板的偏特化，但函数的重载显然是支持的。使用标签分发(Tag Dispatch)的方案就是通过函数实现不同的函数重载实现，根据不同实参类型选择具体的函数实现，以达到函数模板偏特化的实现。
```c++
// 标签分发demo
struct NormalVersionTag {};
struct IntPartialVersionTag {};

template<class T> struct TagDispatchTrait { 
    using Tag = NormalVersionTag; 
};

template<> 
struct TagDispatchTrait<int> { 
    using Tag = IntPartialVersionTag; 
};

template <typename A, typename B>
inline void internal_f(A a, B b, NormalVersionTag) {
    std::cout << "Normal version." << std::endl;
}

template <typename A, typename B>
inline void internal_f(A a, B b, IntPartialVersionTag) {
    std::cout << "Partial version." << std::endl;
}

template <typename A, typename B>
void f(A a, B b) {
    return internal_f(a, b, typename TagDispatchTrait<B>::Tag {});
}

// 测试代码
int a = 10;
double b = 12;

f(a, b);
f(a, a);
```
上述测试代码输出结果为:
```shell
Normal version.
Partial version.
```
可以看到这种方案是利用函数重载的特性以达到根据实参类型筛选不同函数实现的能力。我们将这种实现称为标签分发。

### 2.3 使用Concepts
C++20提供了`Concepts`特性，`Concepts`特性提出的动机是为了解决模板元编程过程中，编译器给出的报错信息冗余及编译器不能很好的给出准确的出错信息的问题。你可以简单的理解为`Concepts`就是在模板元编程过程中需要用户手动打的hints，来帮助编译器知道你在元编程过程中的想法，进而可以更好地给你提供准确的信息。下面看下，如何利用`Concepts`轻松地实现该能力。
```c++
template <typename A, typename B>
void f(A a, B b) {
    std::cout << "Normal version." << std::endl;
}

template <typename A, typename B>
requires std::integral<B>
void f(A a, B b) {
    std::cout << "Partial version." << std::endl;
}
// 测试代码
int a = 10;
double b = 12;
f(a, b);
f(a, a);
```
毫无疑问上述的输出结果还是和之前实现的一样，符合预期。其中对于偏特化的版本其requires B类型为int类型，所以在f(a, a)的调用中，编译器生成且直接匹配到这一个偏特化的版本。不过再次提醒的是，`Concepts`特性是C++20才支持的特性。

## 3. 总结与思考
### 3.1 总结
上述我们提到的三种不同的实现其实都是有各自的优缺点，第一种使用类偏特化的实现优点在于逻辑清晰，传统的C++程序员都能够轻易的理解和实现。第二种使用标签分发的方案实际上是利用函数重载达到函数模板偏特化的效果，实现上有一点绕弯，但`标签分发`的方案是C++标准委员会推荐的一种方法，所以以前在一段时间内开发者所使用的方案。第三种`Concepts`的方案是依赖于C++20，这种方案代码最为简洁和直观，从C++原语上提供了编译器类型要求和类型选择的能力。毫无疑问，未来随着C++20的普及和广泛使用，`Concepts`将是解决这类问题的通用方案。

[https://github.com/jovany-wang/dousi/blob/307426a48d3aeabaf4920325f58d917b326c5096/core/src/core/submitter/service_handle.h#L82](https://github.com/jovany-wang/dousi/blob/307426a48d3aeabaf4920325f58d917b326c5096/core/src/core/submitter/service_handle.h#L82)  
另外这个链接给出了一个使用标签分发实现的函数模板偏特化的实际开发例子。其中调用的`InternalCaller()`时会根据最后一个参数tag进行标签选择不同的实现版本。

### 3.2 思考
通过前面我们了解到函数模板不能直接被偏特化，那么到底为什么标准C++不支持模板函数偏特化呢？简而言之是因为模板特化版本不参与函数的重载抉策过程，因此在和函数重载一起使用的时候，可能出现不符合预期的结果。因此标准C++禁止了函数模板的偏特化。那么有人可能提出疑问，既然C++从语法上就禁止使用函数模板的偏特化，那么为何我们还去做这件事情，岂不是矛盾？其实仔细思考，是不矛盾的。C++禁止的原因是在于函数模板偏特化和函数重载决策的矛盾，而我们在上述的几种实现方案中，都很显式地避开了函数重载的问题。方案1中使用的是类模板偏特化，没有函数重载问题；方案2中使用的就是函数重载本身来作为决策依据；而方案3中，`Concpets`使用在模板函数之上，本身就是利用`Concepts`实现函数的重载，即该过程本身是一个函数重载的决策过程，因此也不存在任何问题。

这里给出一些相关的资料供大家自行思考。  
[C++核心准则: T.144: Don't specialize function templates](https://github.com/isocpp/CppCoreGuidelines/blob/master/CppCoreGuidelines.md#t144-dont-specialize-function-templates)  
[Herb Sutter: Why Not Specialize Function Templates?](http://www.gotw.ca/publications/mill17.htm)  
[A draft proposal Proposed Wording for Concepts.](http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2007/n2307.pdf) 14.5.6.1小节  
[Stack Overflows: Why function template cannot be partially specialized?](https://stackoverflow.com/questions/5101516/why-function-template-cannot-be-partially-specialized)  
[Stack Overflows: Partial specialization of function templates](https://stackoverflow.com/questions/3716799/partial-specialization-of-function-templates)  
