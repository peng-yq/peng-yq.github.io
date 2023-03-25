---
layout: post
title: "【数据结构】二叉搜索树、平衡二叉树"
subtitle: "[Data Structre] Search Binary Tree, Balanced Binary Tree"
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

## Ⅱ. 二叉搜索树

**二叉搜索树：**也称为二叉排序树或二叉查找树

1. 非空**左子树所有键值小于其根结点的键值**。
2. 非空**右子树所有键值大于其根结点的键值**。
3. **左、右子树都是二叉搜索树**。

**二叉搜索树的查找操作：**

1. 从根结点开始，如果树为空，返回NULL。
2. 若树非空，将根结点关键字与查找值比较。
   - 若查找值小于根结点关键字，则在其左子树中继续查找
   - 若查找值大于根结点关键字，则在其右子树中继续查找
   - 若两者比较结果相等，则返回指向此结点的指针

根据上述描述很容易可以想出是用递归的思想，代码如下。

```c++
typedef struct TNode* BinTree;
struct TNode{
    ElementType Data;
    BinTree Left;
    BinTree Right;
};

BinTree Find(BinTree BST, ElementType X){
	if (!BST)
		return  NULL;
	if (X > BST->Data)
		return Find(BST->Right, X);
	else if (X < BST->Data)
		return Find(BST->Right, X);
	else
		return BST;
}
```

若想提高效率，也可将递归换成迭代函数。

```c++
BinTree Find(ElemType x, BinTree BST){
    while(BST){
        if(x > BST->Data)
            BST = BST->Right;
        if(x < BST->Data)
            BST = BST->Left;
        else
            return BST;
    }
    return NULL;
}
```

若查找最大或最小元素，则一直找到最右端或最左端结点，返回其地址即可。

使用递归或迭代的思想均可，下面使用迭代的解法进行演示。

```c++
BinTree FindMax(BinTree BST){
	while (BST){
		if (!BST->Right)
			return BST;
		else
			BST = BST->Right;
	}
	return NULL;
```

**二叉搜索树的插入**

- 二叉树的插入总是插到叶子结点的左孩子或右孩子
- 二叉树的插入关键是找到合适的位置进行插入

```c
BinTree Insert(BinTree BST, ElementType X){
	if (!BST){
		BinTree BST = (BinTree)malloc(sizeof(struct TNode));
		BST->Data = X;
		BST->Left = NULL;
		BST->Right = NULL;
	}
	else{
		if (X < BST->Data)
			BST->Left = Insert(BST->Left, X); 
		else if (X > BST->Data)
			BST->Right = Insert(BST->Right, X); 
	}
	return BST;
}
```

**二叉搜索树的删除需要考虑如下三种情况**

1. 若删除的是叶子结点，直接删除结点，并再修改其父结点指针为NULL
2. 若删除的结点只有一个孩子结点，则将其父节点的指针指向要删除结点的孩子结点，再删除结点
3. 若删除的结点有左右两个孩子结点，则需要使用右子树的最小元素或左子树的最大元素代替被删除结点

```c
BinTree Delete(BinTree BST, ElementType X){
	BinTree Tmp;
	if (!BST)
		printf("Not Found\n");
	else if (X < BST->Data)
		BST->Left = Delete(BST->Left, X);  
	else if (X > BST->Data)
		BST->Right = Delete(BST->Right, X); 
	else{
		if (BST->Left && BST->Right){
			Tmp = FindMin(BST->Right); 
			BST->Data = Tmp->Data;
			BST->Right = Delete(BST->Right, BST->Data); 
		}
		else{
			Tmp = BST;
			if (!BST->Left)
				BST = BST->Right;   
			else if (!BST->Right)
				BST = BST->Left;   		
			free(Tmp);             
		}
	}
	return BST;
}
```

## Ⅲ. 平衡二叉树

**平衡因子：**Balance Factor，BF

$BF(T) = h_L - h_R$，$h_L$表示左子树的高度。

**平衡二叉树：**又称AVL树，为空树或者**任一结点**左、右子树高度差的绝对值不超过1.

结点数为n的平衡二叉树的最大高度为O(log2n)。

**平衡二叉树存在的原因：**我们知道在二叉搜索树中，如果插入元素的顺序接近有序，那么二叉查找树将退化为链表，从而导致二叉查找树的查找效率大为降低。如何使得二叉查找树无论在什么样情况下都能使它的形态最大限度地接近满二叉树以保证它的查找效率呢？这就引出了一种平衡二叉树。

**有关平衡二叉树的调整见[博客](https://blog.csdn.net/leowahaha/article/details/107461619?spm=1001.2101.3001.6661.1&utm_medium=distribute.pc_relevant_t0.none-task-blog-2~default~CTRLIST~default-1.pc_relevant_default&depth_1-utm_source=distribute.pc_relevant_t0.none-task-blog-2~default~CTRLIST~default-1.pc_relevant_default&utm_relevant_index=1)**。

