---
layout: post
title: Effective C++-让自己习惯C++
subtitle: Accustoming Yourself to C++
author: "PYQ"
header-mask: 0.3
mathjax: true
catalog: true
tags:
  - C++
---

## 视C++为一个语言联邦

今天的C++已经是个多重范型编程语言，一个同时支持过程形式、面向对象形式、函数形式、泛型形式、元编程形式的语言。C++主要有4个次语言：

1. C
2. Object-Oriented C++（面向对象）
3. Template C++
4. STL

其中对Template C++的学习和理解主要是为了更深入的去理解诸如STL这种库，毕竟我们大概率不会去接触自己写一个模板，一般项目也不提倡这样做。

当从一个次语言切换到另一个次语言时，高效编程策略会发生变化，例如：

1. 对内置类型（C-like）而言，传值比传引用更高效
2. 对面向对象和泛型编程而言，传引用更高效
3. 而对于STL而言，迭代器和函数对象都是在C指针的基础上塑造出来的，因此传值更高效

- **C++高效编程守则视状况而变化，取决于你使用C++的哪一部分**。

## 尽量以const，enum，inline替换#define

此条款也可以变为“宁可以编译器替换预处理器”。当使用#define定义宏时，在编译前预处理器会对宏进行替换，宏变量名有可能没进入到记号表内（symbol table）。这使得如果编译出错，编译器只会返回定义的值而非变量名，这使得debug会很困难，尤其这个宏出现在非自己写的定义中，我们会花大量的时间去追踪这个错误。

```c++
const char* authorName = "Scott Meyers";  // 也可以写作char const * authorName = "Scott Meyers"; 
const char* const authorName = "Scott Meyers";
```

上面两个const表示不同的意义：

- const在*左边表示authorName指向的内容是不变的，也就是不能通过\*authorName去改变变量的内容；但可以通过改变authorName的值也就是指针的地址去改变变量内容
- const在*右边则表示authorName本身也是不变的，也就不能修改指针的地址去改变变量的内容

相较于使用char*，string往往更合适。

```c++
const string authorName("Scott Meyers");
```

- **对于单纯常量，最好以const或enum替换#define**。
- **对于宏函数，最好改用inline替换**。

## 尽可能使用const

```c++
vector<int> vec;
const vector<int>::iterator iter = vec.begin();co
vector<int>::const_iterator iter = vec.begin();
```

对于STL迭代器，上方两个const的作用不同：

1. 第一个const使得iter指针地址是固定的，但可以改变其指向的内容；也就是不能通过iter++去改变指针，但可以通过*iter去改变值
2. 第二个const_iterator则正好相反

const在面对函数声明时，往往可以降低因客户错误而造成的意外，而又不至于放弃安全性和高效性。

```c++
class Rational {...};
const Rational operator* (const Rational& lhs, const Rational& lhs);
```

const使得不仅使用*的两个变量不会被改变，同时计算所得的值也不会被改变，也就可以避免下面的情况：

```c++
Rational a, b, c;
(a * b) = c;
```

## 确定对象被使用前已先被初始化

通常如果你使用C part of C++而且初始化可能招致运行期成本，那么就不能保证发生初始化（也就是诸如int等内置对象为了提高效率和减少资源消耗，不会对这些对象进行初始化）。而一旦进入non-C parts of C++，规则则会发生变化，诸如vector等对象在没被赋值前会被默认初始化，而array则不会。

对于上述问题，最佳处理办法是**永远在使用对象之前先将其初始化。对于无任何成员的内置类型，需要手动的进行；而内置类型以外的任何其他东西（比如定义的类），初始化由构造函数完成**。值得注意的是需要区分**赋值**和**初始化**，例如下面的代码是在进行赋值而非初始化：除了num（内置类型）外，theName变量会在其构造函数被调用时进行初始化，然后我们在调用Test时对其赋值。这样效率会很低，正确的做法应该是使用**成员初值列**替换赋值动作。

```c++
class Test {
public:
    Test(const string& name);
private:
    string theName;
    int num;
};

// 赋值而非初始化
Test::Test(const string& name){
    theName = name;
    num = 0;
}

// 成员初值列：num变量赋值和初始化成本相同，为了一致性同样使用成员初值列
Test::Test(const string& name)
: theName(name), num(0) {}
```

使用成员初值列时，需要列出所有成员变量，避免漏掉没有初始化的变量。如果成员变量是**const或引用，就一定需要初始值，不能被赋值**。C++有着十分固定的成员初始化次序，类的成员变量总是以其**声明次序被初始化**。**C++对“定义于不同的编译单元内的非本地静态对象”的初始化相对次序并无明确定义，解决办法是将这些非本地静态对象放到专属函数内（通过static来声明，并返回一个指向它的引用），也就是单例模式**。C++保证，函数内的本地静态对象会在“函数被调用期间”“首次遇上该对象定义式”时被初始化，也就是调用该函数时会获得一个被初始化的该对象的引用，而不调用则不会产生构造和析构该对象的成本。

- **手动初始化内置对象**
- **构造函数最好使用成员初值列，排序次序应该和在对象中声明次序相同**
- 使用本地静态对象替换非本地静态对象
