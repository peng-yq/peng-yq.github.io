---
layout: post
title: "「数据结构」静态链表"
subtitle: "「Data Structre」Static Link List"
author: "PYQ"
header-img: "img/post-bg-ds.jpg"
header-mask: 0.3
catalog: true
tags:
  - 数据结构
---

## Ⅰ.前言

之前的数据结构学习中对“静态链表”不太熟悉，最近在阅读《大话数据结构》以及刷算法题时再次遇到“静态链表”，觉得很有意思，对解题的思维有所帮助，遂记录。

## Ⅱ. 静态链表

C/C++以及面向对象语言可以非常容易地操作内存中的地址和数据，而对于早期的编程语言，如何实现链表呢？答案是“静态链表”。

**结构定义**

使用数组代替指针用于表示链表，数组的元素由两个数据域构成，即data（用于存放数据）和cur（相当于链表的next指针，存放下一数据的下标）。

```c
#define ElemType int
#define MAXSIZE 1000

typedef struct{
    ElemType data;
    int cur;
} compoent, SLinkList[MAXSIZE];
```

**初始化**

在静态链表中，对**数组的第一个和最后一个元素作为特殊元素处理，不存放数据。数组的第一个元素的cur（SLinkList[0].cur）存放第一个未被使用结点的下标；数组的最后一个元素的cur（SLinkList[MAXSIZE - 1].cur）存放第一个有数值的元素的下标，扮演头结点的作用。**

```c
void InitSpace_SL(SLinkList &space){
    //将一维数组space中各分量链成一个备用链表，space[0].cur为头指针
    for (int i = 0; i < MAXSIZE - 1;i++){
        space[i].cur = i + 1;
    }
    space[MAXSIZE - 1].cur = 0;
}
```

**插入操作**

在动态链表中，结点的申请和释放分别使用malloc()和free()两个函数实现，在静态链表中操作的是数组，因此需要自己实现。

```c
int Malloc_SL(SLinkList &space){
    //第一个元素cur值即第一个空闲结点下标值
    int i = space[0].cur;
    //由于此时第一个空闲结点在后续要被使用，因此需要更改第一个空闲结点的下标值
    if(space[0].cur)
        space[0].cur = space[i].cur;
    return i;
}
```

```c
int ListInsert(SLinkList L, int i, Elemtype e){
    int j, k, l;
    //先找第一个元素的下标
    k = MAXSIZE - 1;
    if(i<1 || i > ListLength(L)+1)
        return 0;
    j = Malloc_SL(L);
    if(j){
        L[j].data = e;
        for(l=1;l<i-1;l++)
            k = L[k].cur;
        L[j].cur = L[k].cur;
        L[k].cur = j;
        return 1;
    }
    return 0;
}
```

**删除操作**

free()函数的过程如下

1. 被删除的结点成为了第一个可用的结点
2. 修改数组第一个元素的下标值

```c
void Free_SL(SLinkList &space,int k){
    space[k].cur = space[0].cur;
    space[0].cur = k;
}
```

```c
int ListDelete(SLinkList L, int i){
    int j, k;
    if(i<1 || i> ListLength(L))
        return 0;
   	k = MAXSIZE - 1;
    for(j = 1;j <= i-1; j++)
        k = L[k].cur;
    j = L[k].cur;
    L[k].cur = L[j].cur;
    Free_SL(L, j);
    
}
```



