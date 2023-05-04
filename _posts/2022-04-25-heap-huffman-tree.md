---
layout: post
title: "数据结构—堆、哈夫曼树"
subtitle: "[Data Structre] Heap, Huffman tree"
author: "PYQ"
header-img: "img/post-bg-ds.jpg"
header-mask: 0.3
mathjax: true
catalog: true
tags:
  - 数据结构
---

## Ⅰ. 推荐课程

[【浙江大学】数据结构](https://www.bilibili.com/video/BV1Kb41127fT?p=33)，浙大的数据结构讲的很精炼，不枯燥易懂，十分适合数据结构的学习。

以下笔记也是根据以该课程为主，并加以具体实现代码。

## Ⅱ. 堆

**堆：**堆是使用数组实现的**完全二叉树**，其任一结点的值是其子树所有结点的最大值或最小值。

堆可分为大根堆和小根堆：

- **大根堆：**在完全二叉树中，任何一个子树的最大值都在这个子树的根结点。
- **小根堆：**在完全二叉树中，任何一个子树的最小值都在这个子树的根结点。

**注意区别于内存中的堆和栈**，可参考[博客](https://blog.csdn.net/wolenski/article/details/7951961#comments)。

下面代码均使用大根堆进行举例。

**定义堆**

```c
#define MaxData INT_MAX;//定义最大值

typedef int DataType;

typedef struct HeapStruct *MaxHeap;
struct HeapStruct{
	DataType *data;//定义数组
	int size;//堆当前元素个数 
	int capacity;//堆的最大容量 
};
```

**创建大根堆**

注意数组第一个元素不存储堆实际元素，使得编号为i的结点其父结点编号为Int(i/2)，此处将其作为“哨兵”元素，即大根堆即存储一个极大值，使得其大于堆中所有元素，这样在插入删除时就无需与其交换，小根堆则反之。

```c
MaxHeap Creat(int MaxSize){
	MaxHeap H=malloc(sizeof(struct HeapStruct));
	//第一个位置不作为堆的元素 
	H->data=malloc((MaxSize+1)*sizeof(DataType));
	H->size=0;
	H->capacity=MaxSize;
	//第一个元素作为标记元素 
	H->data[0]=MaxData;
	return H;
}
```

**插入操作**

由于堆是完全二叉树，所以插入始终是插入至当前第一个空闲结点位置，此外插入后还需进行调整，即依次与父结点比较，若大于父结点则交换位置。

```c
void Insert(MaxHeap H,DataType item){
	int i;
	if(IsFull(H)){
		printf("最大堆已满！");
		return;
	}
	i=++H->size;
	for(;H->data[i/2]<item;i/=2)
		H->data[i]=H->data[i/2];
	H->data[i]=item;
}
```

**删除操作**

每次删除堆顶元素即最大值或最小值，并使用最后一个元素代替第一个元素，再进行调整。

```c
DataType DeleteMax(MaxHeap H){
	int parent,child;
	DataType MaxItem,temp;
	if(IsEmpty(H)){
		printf("最大堆为空，不存在最大值！");
		return;
	}
	MaxItem=H->data[1];
	//用最大堆中的最后一个元素，从根结点开始向下寻找合适的位置 
	temp=H->data[H->size--];//最后一个元素赋值给temp，然后大小-1 
	
	for(parent=1;parent*2<=H->size;parent=child){
		child=parent*2;
		//寻找儿子结点中最大的 
		if((child!=H->size)&&(H->data[child]<H->data[child])){
			child++;
		}
		//如果temp比两个儿子结点都大，说明找到了合适位置 
		if(temp>=H->data[child])break;
		else
			//temp小于儿子结点中的最大值，那么较大的儿子结点作为parent，返回继续循环 
			H->data[parent]=H->data[child]; 
	}
	//找到合适位置 
	H->data[parent]=temp;
	return MaxItem;
} 
```

## Ⅲ. 哈夫曼树

**哈夫曼树：**又称最优二叉树，是一类带权路径长度最短的树。假设有n个权值{w1,w2,...,wn}，如果构造一棵有n个叶子节点的二叉树，而这n个叶子节点的权值是{w1,w2,...,wn}，则所构造出的带权路径长度最小的二叉树就被称为哈夫曼树。

**树的带权路径长度：**树的带权路径长度指树中所有叶子节点到根节点的路径长度与该叶子节点权值的乘积之和，如果在一棵二叉树中共有n个叶子节点，用Wi表示第i个叶子节点的权值，Li表示第i个也叶子节点到根节点的路径长度，则该二叉树的带权路径长度 WPL=W1 * L1 + W2 * L2 + ... Wn * Ln。

**哈夫曼树的特性：**

- 对于同一组权值，所能得到的哈夫曼树不一定是唯一的
- **哈夫曼树的左右子树可以互换**，因为这并不影响树的带权路径长度
- **带权值的节点都是叶子节点**，不带权值的节点都是某棵子二叉树的根节点
- **权值越大的节点越靠近哈夫曼树的根节点**，权值越小的节点越远离哈夫曼树的根节点
- **哈夫曼树中只有叶子节点和度为2的节点，没有度为1的节点**
- 一棵有n个叶子节点的赫夫曼树共有2n-1个节点

**哈夫曼树的构造：**选取两个最小的值加起来，再从得到的值与剩余的数中选取两个最小的数加起来，直到组成一棵树。

## Ⅳ. 哈夫曼编码

哈夫曼树的应用十分广泛，比如众所周知的哈夫曼编码在通信电文中的应用。在等传送电文时，我们希望电文的总长尽可能短，因此可以对每个字符设计长度不等的编码，让电文中出现较多的字符采用尽可能短的编码。为了保证译码的唯一性，采取左0右1的编码方式。

上图各结点哈夫曼编码如下：

- 1：010
- 2：011
- 3：00
- 4：10
- 5：11

**定义**

```c
typedef struct Node{  
    int weight;                //权值    
    int parent;                //父节点的序号，为-1的是根节点    
    int lchild, rchild;         //左右孩子节点的序号，为-1的是叶子节点    
}HTNode, *HuffmanTree;          //用来存储赫夫曼树中的所有节点   
```

**从根结点开始遍历二叉树求最小带权路径长度**

```c
int countWPL2(HuffmanTree HT, int n){
	int cur = 2 * n - 2;    //当前遍历到的节点的序号，初始时为根节点序号  
	int countRoads=0, WPL=0;//countRoads保存叶子结点的路径长度

//构建好赫夫曼树后，把visit[]用来当做遍历树时每个节点的状态标志  
//visit[cur]=0表明当前节点的左右孩子都还没有被遍历  
//visit[cur]=1表示当前节点的左孩子已经被遍历过，右孩子尚未被遍历  
//visit[cur]=2表示当前节点的左右孩子均被遍历过  
	int visit[maxSize] = { 0 };//visit[]是标注数组,初始化为0

	//从根节点开始遍历，最后回到根节点结束  
	//当cur为根节点的parent时，退出循环  
	while (cur != -1)
	{
		//左右孩子均未被遍历，先向左遍历  
		if (visit[cur]==0)
		{
			visit[cur] = 1;    //表明其左孩子已经被遍历过了  
			if (HT[cur].lchild != -1)
			{   //如果当前节点不是叶子节点，则路径长度+1，并继续向左遍历  
				countRoads++;
				cur = HT[cur].lchild;
			}
			else
			{   //如果当前节点是叶子节点，则计算此结点的带权路径长度，并将其保存起来  
				WPL += countRoads * HT[cur].weight;
			}
		}

		//左孩子已被遍历，开始向右遍历右孩子  
		else if (visit[cur]==1)
		{
			visit[cur] = 2;
			if (HT[cur].rchild != -1)
			{   //如果当前节点不是叶子节点，则记下编码，并继续向右遍历  
				countRoads++;
				cur = HT[cur].rchild;
			}
		}

		//左右孩子均已被遍历，退回到父节点，同时路径长度-1 
		else
		{
			visit[cur] = 0;
			cur = HT[cur].parent;
			--countRoads;
		}

	}
	return WPL;
}
```

