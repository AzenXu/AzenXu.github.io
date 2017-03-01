---
layout: post
title: 听故事学算法 - 和李逍遥一起学「普里姆Prim」算法
---
>数据结构和算法中，求图的最小生成树的普里姆算法，对萌新来说还是有一定压力的。希望这个小故事，能让大家更轻松地把握普里姆算法的思路，为正式学习打好基础。呐，开始咯~

#李大娘生病了
李大娘在余杭镇[盛渔村](http://baike.baidu.com/view/2187559.htm)经营客栈，与侄子李逍遥相依为命，虽然有时言语比较苛刻，但待之如亲子。
一日，客栈来了一群苗人，李大娘病倒，李逍遥拿着黑苗人头目给他的破天锤，踏上了去[仙灵岛](http://baike.baidu.com/view/1311270.htm)的求药之旅。

#仙灵岛有妖怪
想要进入仙灵岛~~偷看到灵儿妹妹洗澡~~给婶婶求药，需要用破天锤打碎岛上的六个雕像。怎奈每条路上都妖怪众多，该怎么办呢？
![仙灵岛地图.png](http://upload-images.jianshu.io/upload_images/163683-75e3e5da136aa7b2.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

#李逍遥行动了
伫立桥头，逍遥望见有三条并不平坦的路，分别通往B、D、G三尊雕像，路上的小怪物分别有5个、11个、3个。逍遥略一思忖，我现在只会气疗术和半吊子的御剑术，英雄不吃眼前亏，姑且忍得一口气，打3个小怪，去砸G雕像吧！
逍遥抡圆了胳膊砸碎G雕像，看到眼前又出现两条路，通往E的路上，有13个妖怪，通往F的路上，有6个妖怪。逍遥凝神细思，喃喃自语道：「从A到B那条路好像只有5个妖怪，不如走那条道吧」。于是逍遥荡起蒲团，从G返回了A，一路杀向了B。
靠着同样的逻辑和惊人的记忆力，逍遥在杀死妖怪数最少的情况下，成功打碎了六个雕像。他走过的路分别是：
- A->G
- A->B
- B->D
- B->C
- C->E
- G->F

![李逍遥路线图.png](http://upload-images.jianshu.io/upload_images/163683-ff50421565456f8a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


他把砸雕塑的故事写成了一封信，附上了自己的路线图，寄给了远房亲戚普里姆大叔，好多年以后，算法和数据结构的试卷上，便多了一道叫做「普里姆算法」的考题。每当看到这道考题，耳边依稀回响起灵儿的低声吟唱：

既不回头，何必不忘；
既然无奈，何须誓言。
今日种种，似水无痕；
明夕何夕，君已陌路。

#逍遥风采依旧
好了，普里姆算法的思路，已经在上面这个故事里了。我们来追寻着逍遥的英姿，一起来通过代码，学习下普里姆算法吧~（PS.以下内容引自[大话数据结构](https://book.douban.com/subject/6424904/)）

**目标：求出下面这个图的最小生成树**

*PS.大家可以任选一个点为起点，用逍遥的思路，尝试着以最小成本连通这9个点，然后跑一下代码，看看结果和自己思考的一不一样*

![普里姆算法实例.png](http://upload-images.jianshu.io/upload_images/163683-5ab14d179851e807.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

**实现代码 - C语言**
```c
#include "stdio.h"    
#include "stdlib.h"   

#include "math.h"  
#include "time.h"

#define OK 1
#define ERROR 0
#define TRUE 1
#define FALSE 0

#define MAXEDGE 20
#define MAXVEX 20
#define INFINITY 65535

typedef int Status;	/* Status是函数的类型,其值是函数结果状态代码，如OK等 */

typedef struct
{
	int arc[MAXVEX][MAXVEX];
	int numVertexes, numEdges;
}MGraph;

void CreateMGraph(MGraph *G)/* 构件图 */
{
	int i, j;

	/* printf("请输入边数和顶点数:"); */
	G->numEdges=15;
	G->numVertexes=9;

	for (i = 0; i < G->numVertexes; i++)/* 初始化图 */
	{
		for ( j = 0; j < G->numVertexes; j++)
		{
			if (i==j)
				G->arc[i][j]=0;
			else
				G->arc[i][j] = G->arc[j][i] = INFINITY;
		}
	}

	G->arc[0][1]=10;
	G->arc[0][5]=11;
	G->arc[1][2]=18;
	G->arc[1][8]=12;
	G->arc[1][6]=16;
	G->arc[2][8]=8;
	G->arc[2][3]=22;
	G->arc[3][8]=21;
	G->arc[3][6]=24;
	G->arc[3][7]=16;
	G->arc[3][4]=20;
	G->arc[4][7]=7;
	G->arc[4][5]=26;
	G->arc[5][6]=17;
	G->arc[6][7]=19;

	for(i = 0; i < G->numVertexes; i++)
	{
		for(j = i; j < G->numVertexes; j++)
		{
			G->arc[j][i] =G->arc[i][j];
		}
	}

}

/* Prim算法生成最小生成树  */
void MiniSpanTree_Prim(MGraph G)
{
	int min, i, j, k;
	int adjvex[MAXVEX];		/* 保存相关顶点下标 */
	int lowcost[MAXVEX];	/* 保存相关顶点间边的权值 */
	lowcost[0] = 0;/* 初始化第一个权值为0，即v0加入生成树 */
			/* lowcost的值为0，在这里就是此下标的顶点已经加入生成树 */
	adjvex[0] = 0;			/* 初始化第一个顶点下标为0 */
	for(i = 1; i < G.numVertexes; i++)	/* 循环除下标为0外的全部顶点 */
	{
		lowcost[i] = G.arc[0][i];	/* 将v0顶点与之有边的权值存入数组 */
		adjvex[i] = 0;					/* 初始化都为v0的下标 */
	}
	for(i = 1; i < G.numVertexes; i++)
	{
		min = INFINITY;	/* 初始化最小权值为∞， */
						/* 通常设置为不可能的大数字如32767、65535等 */
		j = 1;k = 0;
		while(j < G.numVertexes)	/* 循环全部顶点 */
		{
			if(lowcost[j]!=0 && lowcost[j] < min)/* 如果权值不为0且权值小于min */
			{
				min = lowcost[j];	/* 则让当前权值成为最小值 */
				k = j;			/* 将当前最小值的下标存入k */
			}
			j++;
		}
		printf("(%d, %d)\n", adjvex[k], k);/* 打印当前顶点边中权值最小的边 */
		lowcost[k] = 0;/* 将当前顶点的权值设置为0,表示此顶点已经完成任务 */
		for(j = 1; j < G.numVertexes; j++)	/* 循环所有顶点 */
		{
			if(lowcost[j]!=0 && G.arc[k][j] < lowcost[j])
			{/* 如果下标为k顶点各边权值小于此前这些顶点未被加入生成树权值 */
				lowcost[j] = G.arc[k][j];/* 将较小的权值存入lowcost相应位置 */
				adjvex[j] = k;				/* 将下标为k的顶点存入adjvex */
			}
		}
	}
}

int main(void)
{
	MGraph G;
	CreateMGraph(&G);
	MiniSpanTree_Prim(G);

	return 0;

}
```
