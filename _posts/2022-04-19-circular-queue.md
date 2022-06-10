---
layout: post
title: "「数据结构」循环队列"
subtitle: "「Data Structre」Circular Queue"
author: "PYQ"
header-img: "img/post-bg-ds.jpg"
header-mask: 0.3
catalog: true
tags:
  - 数据结构
---

## Ⅰ. 队列的假溢出

队列与生活中的队列很相似，即先进先出。在入队时，只需在队尾增加一个元素，无需移动，即O(1)；而出队时，生活中则是后面的人全部前移一步，这大大增加了时间复杂度。**若假定队头不需要出现在下标为0的位置，这样出队的性能就会大大增加。**

但这也带来了新的问题，即**假溢出**。由于队列中对元素的操作发生在队头和队尾，即入队时尾指针rear+1，出队时头指针front加一，因此会发生如下情况。

![image-20220419191925617](/img/in-post/data-structure-3.png)

此时由于rear=MAXSIZE-1，即最后一个元素，此时再有元素入队则会发生rear越界，而前面还存在两个空闲结点位置，即假溢出。

## Ⅱ. 循环队列

循环队列就是为了解决上述出现的问题，即再从头开始。

![image-20220419193021012](/img/in-post/data-structure-4.png)

为了更好的判断队满，将数组中一个元素“牺牲”，即上图中右边的情况永远不可能出现，左图则表示队满。

**队空：front == rear**

**队满：(rear+1)%Queuesize == front**（rear有可能比front大，也可能比其小，采用取模则可以避免这个问题）

**队列长度：(rear - front + Queuesize) % Queuesize**

## Ⅲ. 实现

**结构**

```c
typedef struct {
    int data[MAXSIZE];      // 队列数据
    int qFront, qRear;      // 队列头、尾指针
}CycleQueue;
```

**初始化**

```c
void CycleQueueInit(CycleQueue *cq)
{
    cq->qFront = 0;
    cq->qRear = 0;
}
```

**判断队是否为空**

```c
int CycleQueueEmpty(CycleQueue *cq)
{
    return cq->qFront == cq->qRear ? 1 : -1;
}
```

**判断队满**

```c
int CycleQueueFull(CycleQueue *cq)      // 区别队列空和满，采取牺牲空间法
{
    return (cq->qRear + 1) % MAXSIZE == cq->qFront ? 1 : -1;
}
```

**入队**

```c
int CycleQueueEnQueue(CycleQueue *cq, int e)
{
    if(CycleQueueFull(cq) > 0)
        return -1;
    cq->data[cq->qRear] = e;
    cq->qRear = (cq->qRear + 1) % MAXSIZE;
    return 1;
}
```

**出队**

```c
int CycleQueueDeQueue(CycleQueue *cq, int *e)
{
    if(CycleQueueEmpty(cq) > 0)
        return -1;
    *e = cq->data[cq->qFront];
    cq->qFront = (cq->qFront + 1) % MAXSIZE;
    return 1;
}
```

