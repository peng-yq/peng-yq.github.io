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

**C++高效编程守则视状况而变化，取决于你使用C++的哪一部分**。

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

**对于单纯常量，最好以const或enum替换#define**。

**对于宏函数，最好改用inline替换**。

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

