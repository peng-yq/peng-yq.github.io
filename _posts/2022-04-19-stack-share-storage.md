---
layout: post
title: "「数据结构」两栈共享存储空间"
subtitle: "「Data Structre」Two Stacks Share Storage Space"
author: "PYQ"
header-img: "img/post-bg-ds.jpg"
header-mask: 0.3
catalog: true
tags:
  - 数据结构
---

## Ⅰ. 栈的应用

1. 子程序的调用
2. 递归
3. 进制转换
4. 表达式的转换与求值（后缀表达式）
5. 二叉树的便利
6. 图的深度优先算法
7. 浏览器的返回
8. 软件的撤销操作

## Ⅱ. 两栈共享存储空间

假设有两个同类型的栈，各自有各自的数组空间，当一个栈已经满了，再进栈就会发生溢出，而另外一个栈还有很多存储空间，此时为了更好的利用存储空间，我们可以设计两个栈共享一个存储空间。

![image-20220419165454038](/img/in-post/data-structure-2.png)

**定义**

1. 让一个栈的栈底为数组的始端，即下标0处
2. 让另外一个栈的栈底为数组的末端，即数组长度n-1处
3. 两个栈若增加元素，栈顶向中间靠拢

**栈满与栈空**

1. 栈1为空：top1 = -1
2. 栈2为空：top2 = n
3. 栈满：top1 +1 = top2

**结构**

```c
typedef struct{
    SElemType data[MAXSIZE];
   	int top1;
    int top2;
}SqDoubleStack;
```

**入栈**

插入新元素时需要有一个判断插入栈编号的参数。

```c
int Push(SqDoubleStack *S, SElemType e, int stackNumber){
    //先判断是否栈满
    if(S->top1 +1 == S->top2)
        return 0;
    if(stackNumber == 1)
        S->data[++S->top1]=e;
    else if(stackNumber == 2)
        S->data[--S->top2]=3;
    return 1;
}
```

**出栈**

```c
int Pop(SqDoubleStack *S, SElemType *e, int stackNumber){
    if(stackNumber==1){
        if(S->top1==-1)
            return 0;
        *e=S->data[S->top1--];
    }
    else if(stackNumber==2){
        if(S->top2==MAXSIZE)
            return 0;
        *e=S->data[S->top2++];
    }
    return 1;
}
```

**场景**

通常用于**两个同类型栈的空间需求有相反关系时**，比如买卖。