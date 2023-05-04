---
layout: post
title: "数据结构—Prime算法和Kruskal算法"
subtitle: "[Data Structre] Prime, Kruskal"
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

## Ⅱ. 最小生成树

Prime算法和Kruskal算法均是用于解决最小生成树问题的算法，比如如何使用最小的权值（路径或者花费）将多个地点进行连通就是一个最小生成树问题。

**生成树：**树其实是一种特殊的图，一个连通图的生成树是一个极小的**连通子图**，它包含图中全部的**n个顶点**，但只有构成一棵树的**n-1条边**。

**生成树的特性：**

- 一个连通图可以有多个生成树
- 一个连通图的所有生成树都包含相同的顶点个数和边数
- 生成树当中**不存在环**
- **移除生成树中的任意一条边都会导致图的不连通， 生成树的边最少特性**
- 在生成树中**添加一条边会构成环**
- 对于包含n个顶点的连通图，生成树包含n个顶点和n-1条边

一个连通图可以有多个生成树，其中加权最小的树为**最小生成树**。

## Ⅲ. Prime算法

Prime算法本质是一个贪心算法，其基本思想如下：

1. 清空生成树，任取一个顶点加入生成树
2. 在那些一个端点在生成树里，另一个端点不在生成树里的边中，选取一条权最小的边，将它和另一个端点加进生成树
3. 重复步骤2，直到所有的顶点都进入了生成树为止，此时的生成树就是最小生成树

即从连通网络 N = { V, E }中的某一顶点 u0 出发，选择与它关联的具有最小权值的边(u0, v)，将其顶点v加入到生成树的顶点集合U中。以后每一步从一个顶点在U中，而另一个顶点不在U中的各条边中选择权值最小的边(u, v),把它的顶点 v加入到集合U中。如此继续下去，直到网络中的所有顶点都加入到生成树顶点集合U中为止。

具体视频讲解见浙江大学数据结构视频。

> 原理很简单，代码写起来是真的复杂，但其实需要使用的是小根堆和并查集（前者取出最小的，后者避免形成环）

```c
#include <stdio.h>
#include <stdlib.h>
#define VertexMax 20 //最大顶点数为100
#define MaxInt 32767 //表示最大整数，表示 ∞ 

typedef char VertexType; //每个顶点数据类型为字符型 

//邻接矩阵结构体 
typedef struct{
	VertexType Vertex[VertexMax];//存放顶点元素的一维数组 
	int AdjMatrix[VertexMax][VertexMax];//邻接矩阵二维数组 
	int vexnum,arcnum;//图的顶点数和边数 
}MGraph;

//辅助数组结构体(候选最短边) 
typedef struct{
	VertexType adjvex;//候选最短边的邻接点 
	int lowcost;//候选最短边的权值 
}ShortEdge;

//查找元素v在一维数组 Vertex[] 中的下标，并返回下标 
int LocateVex(MGraph *G,VertexType v){
	int i;	
	for(i=0;i<G->vexnum;i++){
		if(v==G->Vertex[i]){
			return i; 
		} 
	 } 	 
	 printf("No Such Vertex!\n");
	 return -1;
}

//构建无向网（Undirected Network）
void CreateUDN(MGraph *G){
	int i,j;
	//1.输入顶点数和边数
	printf("输入顶点个数和边数：\n");
	printf("顶点数 n="); 
	scanf("%d",&G->vexnum);
	printf("边  数 e="); 
	scanf("%d",&G->arcnum);
	printf("\n"); 	
	printf("\n");
	
	//2.输入顶点元素 
	printf("输入顶点元素(无需空格隔开)：");
	scanf("%s",G->Vertex);
	printf("\n");
    
	//3.矩阵初始化
	for(i=0;i<G->vexnum;i++){
       for(j=0;j<G->vexnum;j++){
	    	G->AdjMatrix[i][j]=MaxInt;
		} 
    } 
	 	
	 //4.构建邻接矩阵
	 int n,m;
	 VertexType v1,v2;
	 int w;//v1->v2的权值 
	 
	 printf("请输入边的信息和权值(例：AB,15)：\n");
	 for(i=0;i<G->arcnum;i++){
	 	printf("输入第%d条边信息及权值：",i+1);
	 	scanf(" %c%c,%d",&v1,&v2,&w);
	 	n=LocateVex(G,v1); //获取v1所对应的在Vertex数组中的坐标 
	 	m=LocateVex(G,v2); //获取v2所对应的在Vertex数组中的坐标	 	
	 	if(n==-1||m==-1){
		 	printf("NO This Vertex!\n");
		 	return;
		  } 	
	   G->AdjMatrix[n][m]=w;
	   G->AdjMatrix[m][n]=w;//无向网仅此处不同 
     } 
}

void print(MGraph G){
	printf("\n-------------------------------");
	printf("\n 邻接矩阵：\n\n"); 	
    for(int i=0;i<G.vexnum;i++){
        printf("\t%c",G.Vertex[i]);
        for(int j=0;j<G.vexnum;j++){
            if(G.AdjMatrix[i][j]==MaxInt)
                printf("\t∞");
            else printf("\t%d",G.AdjMatrix[i][j]);
        }
        printf("\n");
    }	 
}

int minimal(MGraph *G,ShortEdge *shortedge){
	int i,j;
	int min,loc;	
	min=MaxInt;
	for(i=1;i<G->vexnum;i++){
		if(min>shortedge[i].lowcost&&shortedge[i].lowcost!=0){
			min=shortedge[i].lowcost;
			loc=i;
		}
	}
	return loc;
}
 
void MiniSpanTree_Prim(MGraph *G,VertexType start)
{ 
	int i,j,k;
	ShortEdge shortedge[VertexMax];
	
	//1.处理起始点start 
	k=LocateVex(G,start);
    //初始化其他点与起始点的距离
	for(i=0;i<G->vexnum;i++){
		shortedge[i].adjvex=start;
		shortedge[i].lowcost=G->AdjMatrix[k][i];
	}
	shortedge[k].lowcost=0;//lowcost为0表示该顶点属于U集合 
	
	//2.处理后续结点
    //对集合U，去找最短路径的顶点 
	for(i=0;i<G->vexnum-1;i++){
		k=minimal(G,shortedge);//找最短路径的顶点 
	    printf("%c->%c,%d\n",shortedge[k].adjvex,G->Vertex[k],shortedge[k].lowcost);//输出找到的最短路径顶，及路径权值 
	    shortedge[k].lowcost=0;//将找到的最短路径顶点加入集合U中 
        //U中加入了新节点，可能出现新的最短路径，故更新shortedge数组
	    for(j=0;j<G->vexnum;j++) {
            //有更短路径出现时，将其替换进shortedge数组 
	    	if(G->AdjMatrix[k][j]<shortedge[j].lowcost){
	    		shortedge[j].lowcost=G->AdjMatrix[k][j];
	    		shortedge[j].adjvex=G->Vertex[k];
			}
		}
	    
	 } 
}

int main() {
	VertexType start;	 
	MGraph G;
	CreateUDN(&G);
	print(G); 	
	printf("请输入起始点：");
	scanf(" %c",&start);//%c前面有空格 
	MiniSpanTree_Prim(&G,start);	 
	return 0;
}
```

## Ⅳ. Kruskal算法

Kruskal算法是另一种贪心算法，我们将图中的每个edge按照权重大小进行排序，每次从边集中取出权重最小且两个顶点都不在同一个集合的边加入生成树中！注意：如果这两个顶点都在同一集合内，说明已经通过其他边相连，因此如果将这个边添加到生成树中，那么就会形成环！这样反复做，直到所有的节点都连接成功！Prime算法是对点进行操作，而Kruskal算法则是对边进行操作。

```C
#include <stdio.h>
#include <stdlib.h>
#define VertexMax 20 //最大顶点数为20

typedef char VertexType; 

typedef struct{
	VertexType begin;
	VertexType end;
	int weight;
}Edge;//边集数组edge[]的单元 

typedef struct{
	VertexType Vertex[VertexMax];//顶点数组 
	Edge edge[VertexMax];//边集数组 
	int vexnum;//顶点数 
	int edgenum;//边数 
}EdgeGraph;

void CreateEdgeGraph(EdgeGraph *E){
	int i;	
	printf("请输入顶点数和边数:\n");
	printf("顶点数 n="); 
	scanf("%d",&E->vexnum);
	printf("边  数 e="); 
	scanf("%d",&E->edgenum);
	printf("\n"); 	
	printf("输入顶点(无需空格隔开):"); 
	scanf("%s",E->Vertex);
	printf("\n\n");
	printf("输入边信息和权值(如：AB,15):\n");
	for(i=0;i<E->edgenum;i++){
		printf("请输入第%d边的信息:",i+1);
		scanf(" %c%c,%d",&E->edge[i].begin,&E->edge[i].end,&E->edge[i].weight);
	}	
}

void print(EdgeGraph *E){
	int i;	
	printf("\n-----------------------------------\n"); 
	printf(" 顶点数组Vertex:");
	for(i=0;i<E->vexnum;i++){
		printf("%c ",E->Vertex[i]);
	}
	printf("\n\n");	
	printf(" 边集数组edge:\n\n");
	printf("\t\tBegin	End	Weight\n");
	for(i=0;i<E->edgenum;i++)	{
		printf("\tedge[%d]	%c	%c	%d\n",i,E->edge[i].begin,E->edge[i].end,E->edge[i].weight);
	}
	printf("\n-----------------------------------\n");
}

//冒泡排序
void sort(EdgeGraph *E){
	int i,j;
	Edge temp;	
	for(i=0;i<E->edgenum-1;i++){
		for(j=i+1;j<E->edgenum;j++){
			if(E->edge[i].weight>E->edge[j].weight){
				temp=E->edge[i];
				E->edge[i]=E->edge[j];
				E->edge[j]=temp;
			}
		}
	}
}

int LocateVex(EdgeGraph *E,VertexType v){
	int i;	
	for(i=0;i<E->vexnum;i++){
		if(v==E->Vertex[i])
			return i; 
	 } 	 
	 printf("No Such Vertex!\n");
	 return -1;
}

//并查集
int FindRoot(int t,int parent[]){
	while(parent[t]>-1)//parent=-1表示没有双亲，即没有根节点 
		t=parent[t];//逐代查找根节点 	
	return t;//将找到的根节点返回，若没有根节点返回自身 
}

void MiniSpanTree_Kruskal(EdgeGraph *E){
	int i;
	int num;//生成边计数器，当num=顶点数-1 就代表最小生成树生成完毕 
	int root1,root2; 
	int LocVex1,LocVex2; 
	int parent[VertexMax];//用于查找顶点的双亲，判断两个顶点间是否有共同的双亲，来判断生成树是否会成环 
	
	//1.按权值从小到大排序 
	sort(E);
	print(E);
	//2.初始化parent数组 
	for(i=0;i<E->vexnum;i++)
		parent[i]=-1;	
	printf("\n 最小生成树(Kruskal):\n\n");
	//3.
	for(num=0,i=0;i<E->edgenum;i++){
		LocVex1=LocateVex(E,E->edge[i].begin);
		LocVex2=LocateVex(E,E->edge[i].end);
		root1=FindRoot(LocVex1,parent);
		root2=FindRoot(LocVex2,parent);
        //若不会成环，则在最小生成树中构建此边 
		if(root1!=root2){
			printf("\t\t%c-%c w=%d\n",E->edge[i].begin,E->edge[i].end,E->edge[i].weight);//输出此边 
			parent[root2]=root1;//合并生成树
			num++;			
			if(num==E->vexnum-1)
				return;
		} 
	}
	
}

int main() {
	EdgeGraph E;
	
	CreateEdgeGraph(&E);
	MiniSpanTree_Kruskal(&E);
	
	return 0;
	
}
```

