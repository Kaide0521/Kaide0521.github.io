---
title: LCA算法详解
date: 2020-01-05 21:18:57
comments: true
toc: true
categories:
- 算法练习
tags: 
- 最近公共祖先

---
##  LCA算法详解
### 1. 概述
LCA（Least Common Ancestors），即最近公共祖先，是指这样一个问题：在有根树中，找出某两个结点u和v最近的公共祖先（另一种说法，离树根最远的公共祖先）。对于该问题，最容易想到的解决方案是遍历，复杂度是O(n)。但当数据量非常大且查询很频繁时，该算法也许会存在问题。
### 2. 在线ST算法
解决此问题存在两种经典的算法，一种是在线ST算法，另外一种是离线的Tarjan算法，
所谓在线算法是指用户每输入一个查询便马上处理一个查询，该算法一般用较长的时间做预处理，待信息充足以后便可以用较少的时间回答每个查询。
所谓离线算法，是指首先读入所有的询问（求一次LCA叫做一次询问），然后重新组织查询处理顺序以便得到更高效的处理方法
在线算法DFS+ST描述：将树看成一个无向图，u和v的公共祖先一定在u与v之间的最短路径上

  ● DFS：从树的根节点T开始，深度遍历树，并记录下每次到达的顶点。第一个的结点是root(T)，每经过一条边都记录它的端点。由于每条边恰好经过2次，因此一共记录了2n-1个结点，用E[1, ... , 2n-1]来表示。
  
  ● 计算R[]：用R[i]表示E数组中值为i的元素第一次出现的下标，即如果R[u] < R[v]时，DFS访问的顺序是E[R[u], R[u]+1, …, R[v]]。虽然其中包含u的后代，但深度最小的还是u与v的公共祖先。
  
  ● RMQ：当R[u] ≥ R[v]时，LCA[T, u, v] = RMQ(L, R[v], R[u])；否则LCA[T, u, v] = RMQ(L, R[u], R[v])，计算RMQ。

#### 在线算法DFS+ST代码如下：

##### 1.数据结构描述：
```
    
    // 头结点信息
	private Node heads[];
	// 边信息
	private Edge edges[];
	// 深度遍历过程中每个节点第一次出现的序号
	private int first[];
	// 每个节点出现的深度
	private int depth[];
	// 深度遍历序列
	private int travel[];
	// 保存节点到根节点的距离
	private int dir[];
	// 访问记录矩阵
	private boolean vis[];
	private RMQ mRmq;

	/**
	 * 邻接表头结点信息
	 */
	class Node {
		private int sno;// 节点编号
		private Edge firstEdge;
	}

	/**
	 * 邻接表边信息
	 */
	class Edge {
		private int sno;
		private int from;
		private int to;
		private int wight;
		private Edge next;
	}

```
##### 1.算法步骤描述：

```
#####     1.根据输入的节点和权重信息建立邻接表
    /**
	 * 无向图创建路径
	 * 
	 * @param from
	 * @param to
	 * @param wight
	 */
	public void createEdge(int from, int to, int wight) {
		addEdge(from, to, wight);
		addEdge(to, from, wight);
	}

	/**
	 * 头插法创建邻接表
	 * 
	 * @param from
	 * @param to
	 * @param wight
	 */
	private void addEdge(int from, int to, int wight) {
		Edge edge = new Edge();
		edge.from = from;
		edge.to = to;
		edge.wight = wight;
		edge.sno = edgeNum;
		edges[edgeNum++] = edge;
		edge.next = heads[from].firstEdge;
		heads[from].sno = from;
		heads[from].firstEdge = edge;
	}
	
#####     2.深度优先遍历，计算访问序列、深度信息、节点第一次出现的位置信息等
    
    /**
	 * 深度遍历邻接表
	 * 
	 * @param u
	 * @param dep
	 */
	public void travelInDepth(int u, int dep) {
		vis[u] = true;
		travel[++index] = u;
		first[u] = index;
		depth[index] = dep;
		for (Edge edge = heads[u].firstEdge; edge != null; edge = edge.next) {
			if (!vis[edge.to]) {
				int v = edge.to;
				dir[v] = dir[u] + edge.wight;
				travelInDepth(v, dep + 1);
				travel[++index] = u;
				depth[index] = dep;
			}
		}
	}
	
#####     3.根据输入的查询，回答结果
    /**
	 * 回答节点的最近公共祖先节点
	 * @param u
	 * @param v
	 * @return
	 */
	public int getLCANode(int u, int v) {
		if (mRmq == null) {
			mRmq = new RMQ();
			mRmq.RMQInit(depth);
		}
		u = first[u];
		v = first[v];
		if (u < v) {
			return mRmq.getMax(u, v);
		}
		return mRmq.getMax(v, u);
	}


```