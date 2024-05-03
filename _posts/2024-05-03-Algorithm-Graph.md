---
layout: post
title: Algorithm-Graph(Basic)(Ongoing)
subtitle: picture from https://www.pexels.com/search/wild%20animals/
author: maxshuang
categories: Algorithm
banner:
  image: /assets/images/post/algorithm-graph/hive.jpg
  opacity: 0.618
  background: "#000"
  height: "55vh"
  min_height: "38vh"
  heading_style: "font-size: 3.00em; font-weight: bold; text-decoration: underline"
  subheading_style: "color: gold"
tags: Algorithm Graph
---

> NOTE: 内容来自于 [算法4](https://algs4.cs.princeton.edu/home/), 主要记录个人对其中算法的理解。有兴趣或者需要 Java 源码可以直接阅读 [online website graph](https://algs4.cs.princeton.edu/40graphs/). Enjoy it:)

## 图

图结构是现实世界的一个重要抽象，表达的是顶点间关联关系，广泛存在于多种场景中，比如：
* 地图：非常直接的图关系，地点是顶点，道路是路径。
* 人际关系和社交网络：每个人有多个好友，好友又有好友，可能存在环状的相识链，好友关系一般是双向关系，而微博等平台的关注关系则是单向关系。
* 网站超链接：网站作为经典的分布式应用，网站内部通过维护其他网站超链接的方式组成一个超大规模图。
* 商品调度/贸易关系/软件设计：这些场景都涉及了多个模块之间的交互，自然存在顶点和连接的图抽象。

经典的图分类可以分成无向图和有向图：
![undirected-graph](/assets/images/post/algorithm-graph/classic-graph.png)
![directed-graph](/assets/images/post/algorithm-graph/classic-directed-graph.png)

## 图的性质

图由顶点和边构成，存在以下性质和相关概念：  
![attributes](/assets/images/post/algorithm-graph/anatomy-of-a-graph.png)
1. 顶点度(degree)/入度(in-degree)/出度(out-degree)：直接连接到该顶点的边个数称为顶点的度，在有向图中，度被进一步区分成入度和出度。
2. 路径(path)/路径长度(length of path)：边连接的一系列顶点称为路径，路径长度为边的个数。
3. 联通(connected)/联通分量(connected component): 当每个顶点对其他顶点都存在至少一条路径时我们称图是联通的；联通分量表示图中互相联通的顶点集合；一个图中可能含有多个联通分量。
4. 无环图(acyclic graph): 一个图中没有环则称为无环图，是一颗树。
![acyclic-graph](/assets/images/post/algorithm-graph/acyclic-graph.png)
5. 生成树(spanning tree)：一个图中连接所有顶点的树称为生成树，生成树恰好有 V-1 条边，且添加任意一条额外的边都会在树中成环。比如下图中阴影部分就是各个联通分量的生成树，生成树在研究*图连通性*和*最小连通问题*等问题领域都有重要作用。
![spanning-tree-forest](/assets/images/post/algorithm-graph/spanning-tree-forest.png)
6. 森林(forest)和生成树森林(spanning forest): 多个树组成森林，多个生成树组成生成树森林。

## 图的表示和操作

图在大的分类上分成无向图和有向图，无向图的边是双向的，有向图的边是单向的。

在下面的表示中，为了结构清晰，参考[算法4](https://algs4.cs.princeton.edu/home/)我们会将图的表示分成图行为和图实现，并且分离出图搜索等相关算法。其中：
* 图行为：提供一个图的基本操作，比如顶点个数，边个数，特点顶点的顶点集等。
* 图实现：具体实现图的方法，比如是使用数组还是链表存储顶点和边集合。
* 图操作：包括图搜索等算法，每个操作可以独立开来，只和图行为交互，和图表示没有关系。

在描述图和图算法过程中，我们使用面向对象的语言 C++，面向对象的设计方法可以有效实现隐藏相关实现，并且通过 public 接口定义可以明确各组件的交互边界。

### 图的操作

无向图中所有顶点间连接都是无向的。图行为定义如下：
```
/**
 * @struct Edge
 * @brief Represents an edge in the graph.
 */
struct Edge
{
    //
    // behavior
    //
    Edge(int s, int d, double w=0.0);
    int Src() const;
    int Dest() const;
    double Weight() const;
    bool operator==(const Edge& other) const;
};

/**
 * @struct Graph
 * @brief Represents an undirected graph without edge weights.
 */
class Graph
{
    //
    // behavior
    //
public:
    typedef ConstIterator2<Edge> const_iterator;
    UndirectedGraph(int V);
    ~UndirectedGraph();
    void AddEdge(int s, int t, double w = 0);
    int V() const;
    int E() const;
    // Adj() returns a pair of const iterator [begin, end) for the adjective vertices list
    std::pair<const_iterator, const_iterator> Adj(int v) const;
    // Edges() returns a pair of const iterator [begin, end) for the whole edge list
    std::pair<const_iterator, const_iterator> Edges();
};
```

### 图的表示
[TODO]

### 图的深度优先遍历(DFS)和宽度优先遍历(BFS)

图的深度遍历是指优先访问路径中的 succesor vertice, 相当于多叉树的后序遍历。图的宽度优先遍历是指优先访问当前顶点的所有邻接顶点，再访问二级邻接顶点，类似 Fan-out 的访问模式。

#### 深度优先遍历(DFS)

DFS 可以依赖递归很好得实现顶点回溯访问，同时为了避免重复访问相同的顶点，需要标记数组 $marked$ 标记顶点以被访问过。[完整实现版本链接](https://github.com/maxshuang/Demo/blob/main/algorithm/algorithm/graph/undirected_graph/dfs_path.cpp)。
```
void DfsPaths::dfs(const UndirectedGraph& G, int v) {
    this->marked_[v] = true;
    auto pair = G.Adj(v);
    std::for_each(pair.first, pair.second, [&, v](const Edge& w){
        if (!marked_[w.Dest()]) {
            edge_to_[w.Dest()] = v;
            dfs(G, w.Dest());
        }
    });
}
```

DFS Time Complexity 为 $O(E+V)$，每个边都被访问一次(无向边可以认为是双向的有向边)，顶点也被访问一次(这里特指被标记一次，整体访问顶点次数只有系数差别)。Space Complexity 为 $O(V)$。

#### 宽度优先遍历(BFS)

BFS 需要借助队列实现 FIFO 的效果，每访问一个顶点都优先将它的所有未被访问过的邻接顶点入队列。[完整实现版本链接](https://github.com/maxshuang/Demo/blob/main/algorithm/algorithm/graph/undirected_graph/bfs_path.cpp)。
```
void BfsPaths::bfs(const UndirectedGraph& G, int v) {
    std::queue<int> q;
    this->marked_[v] = true;
    q.push(v);
    while (!q.empty()) {
        // for a queue, we use front() to get the first element and pop() to remove it
        // to avoid "pop" throw out some exceptions 
        int x = q.front();
        q.pop();
        auto pair = G.Adj(x);
        std::for_each(pair.first, pair.second, [&,x](const Edge& w){
            if (!marked_[w.Dest()]) {
                edge_to_[w.Dest()] = x;
                // mark the vertice before pushing to the queue
                marked_[w.Dest()] = true;
                q.push(w.Dest());
            }
        });
    }
}
```
BFS Time Complexity 为 $O(E+V)$，每个边都被访问一次(无向边可以认为是双向的有向边)，顶点也被访问一次(这里特指被标记一次，整体访问顶点次数只有系数差别)。Space Complexity 为 $O(V)$。

### 无向图

无向图的边是无向的，也可以认为是双向的。它是对双向关系的建模，比如电路系统中电线连接的 2 个组件，道路中的双向通道和社交中的朋友关系等等。

#### 环检测

环检测是图的一个常见应用，使用的方法也非常直接 DFS。DFS 本质上是多叉树的回溯访问，在环检测中，我们可以维护当前访问的 path，如果发现邻接顶点已经在 path 中，说明当前 path 成环了。

无向图在环检测实现中要注意排除掉起始顶点，避免误判，比如边<2, 3>，DFS 从 2 访问到 3，3 可能又沿着原始边 <2, 3> 访问到 2，导致误判。[完整实现版本链接](https://github.com/maxshuang/Demo/blob/main/algorithm/algorithm/graph/undirected_graph/cycle.hpp)。
```
void dfs(const UndirectedGraph &g)
{
    for (int i = 0; i < g.V(); ++i)
    {
        if (!has_cycle_ && !marked_[i])
        {
            // source and destination is itself
            dfs_recur(g, i, i);
        }
    }
}
void dfs_recur(const UndirectedGraph &g, int v, int from)
{
    if (has_cycle_)
        return;

    // add v to path
    edge_to_[v] = from;
    marked_[v] = true;

    auto pair = g.Adj(v);
    std::for_each(pair.first, pair.second, [&](const Edge &e)
                    {
        if(has_cycle_) return;

        if(!marked_[e.Dest()]) { 
            dfs_recur(g, e.Dest(), e.Src());
        }else if(from != e.Dest()){
            // because edge of undirected graph is bidirected, 
            // so if we find out this edge is not the original edge, 
            // we know it has a cycle.
            has_cycle_=true;
            // construct a cycle
            edge_to_[e.Dest()]=e.Src();
            get_cycle(e.Src());
        } });

    // revert back
    edge_to_[v] = -1;
}
```

环检测时间 Time Complexity 为 $O(E+V)$，Space Complexity 为 $O(V)$。

#### 连通分量(Connected Component)

无向图另外一个有意思的性质是连通性 connectivity，因为无向图的边是没有方向性的，所以只要存在边连接的顶点都是互相 connected，我们称为连通分量(Connected Component)，比如下图中就存在 3 个连通分量。
![undirected-graph](/assets/images/post/algorithm-graph/classic-graph.png)

一次 DFS 就可以访问到一个连通分量中的所有顶点，所以只要对图中逐个顶点进行 DFS 即可知道图中有多少个 Connected Component，当然已经访问过的顶点就不用再运行 DFS 了。[完整实现版本链接](https://github.com/maxshuang/Demo/blob/main/algorithm/algorithm/graph/undirected_graph/connected_component.hpp)。
```
void ConnectedComponent::dfs(const UndirectedGraph &G)
{
    // run dfs for all vertices, but skip those which have been visited
    for (int i = 0; i < G.V(); ++i)
    {
        if (!marked_[i])
        {
            ++count_;
            dfs_recur(G, i);
        }
    }
}

void ConnectedComponent::dfs_recur(const UndirectedGraph &G, int v)
{
    marked_[v] = true;
    index_[v] = count_;

    auto pair = G.Adj(v);
    std::for_each(pair.first, pair.second, [&](const Edge &w)
                  {
        if(!marked_[w.Dest()]) {
            dfs_recur(G, w.Dest());
        } });
}
```

检测连通分量其实本质上和对一个连通图运行一次 DFS 没有区别，时间 Time Complexity 为 $O(E+V)$，Space Complexity 为 $O(V)$。

### 有向图

有向图的边是单向的，只能起始顶点到结束顶点，没有反向性质，所以有向图研究的是可达性 reachability，不是 connectivity。它是单向关系的建模，比如社交网络中粉丝对偶像的单向关注，职场环境中下级对上级的单向汇报关系等。

#### 环检测和 DAG(Directed Acyclic Graph)

有向图的环检测和无向图一样，都是通过为当前 path 判断是否出现了 path 回环。无环有向图被称为 DAG(Directed Acyclic Graph), DAG 被广泛应用于任务调度中，因为 DAG 可以获得一个满足所有 precedence contraint 的拓扑排序，使得任务调度可以依序进行。[完整实现版本链接](https://github.com/maxshuang/Demo/blob/main/algorithm/algorithm/graph/directed_graph/dag.hpp)。

```
void dfs(const Digraph &G)
{
    for (int i = 0; i < G.V() && !has_cycle_; ++i)
    {
        if (!marked_[i])
        {
            dfs_recur(G, i, i);
        }
    }
}

void dfs_recur(const Digraph &G, int v, int from)
{
    if(has_cycle_) return;

    marked_[v] = true;
    edge_to_[v]=from;
    auto pair = G.Adj(v);
    std::for_each(pair.first, pair.second, [&](const Edge &e)
                    {
    if(has_cycle_) return;

    if(!marked_[e.Dest()]) {
        dfs_recur(G, e.Dest(), e.Src());
    }else if(edge_to_[e.Dest()]!=-1){
        // hit the cycle
        has_cycle_=true;
        // construct the cycle
        edge_to_[e.Dest()]=e.Src();
        get_cycle(e.Src());
    } });

    edge_to_[v]=-1;
}
```
有向图环检测也是类似运行一次 DFS，时间 Time Complexity 为 $O(E+V)$，Space Complexity 为 $O(V)$。

#### DAG 和拓扑排序(Topological Sort)

DAG 是对无循环依赖的关系的一个抽象，比如多个任务之间互相依赖，需要研究如何安排任务的开始时间以满足所有的依赖关系，我们称这个问题为前置依赖调度(Precedence-constrained scheduling)。
![precedence constraints](/assets/images/post/algorithm-graph/precedence_constraints.png)
更形象的，可以看成如何排列顶点，使得所有的有向边都朝向一个方向，从而满足所有的 precedence constraints，称这种次序为拓扑序(topological sort)。
![topological sort](/assets/images/post/algorithm-graph/topological_sort.png)

获取 topological sort 的难点在于无法捕捉一个顶点的所有邻接顶点间的次序关系，起码一次 DFS pre-order 只能获取到父子顶点的次序关系，而无法获得 sibling 顶点的次序关系。举个例子：
```
<1, 2>
<1, 3>
<2, 3>
<3, 4>

pre-order DFS: 1 -> 3 -> 4 -> 2
we need topological sort: 1 -> 2 -> 3 -> 4
```

为了解决这个问题，捕捉一个顶点的所有邻接顶点间的次序关系，我们可以观察 DFS 的一个很有意思的特性，就是它的 post-order traversal。举个例子：
```
<A, B>
<A, C>
<B, C>
<C, D>
```
对于这样一个 A 的邻接顶点间存在有向关系的图，post-order traversal 输出结果是从后往前输出的，所以:
1. 如果先访问到 <A, B>，则一定会继续访问 <B, C>， <C, D>，此时有向关系导致顶点的 post-order 输出为 {D, C, B, A};
2. 如果先访问到 <A, C>, 则一定会继续访问 <C, D>, 再回溯访问 <B, C>， 此时 post-order 输出为 {D, C, B, A};

他们的 reverse post order 都为 {A, B, C, D}，*貌似 reverse post-order traversal 能在一次 DFS 后捕获到一个顶点的所有邻接顶点间的次序关系*。

*为什么呢?*

我没法准确描述这里的原因。只能从观察上看到，如果先访问到 precedence vertex，则 DFS 本身就可以保证 partial topological sort。如果先访问到 successor vertex，回溯算法本身也可以保证 precedence vertex 会被后访问，仍然保证 partial topological sort。

[完整实现版本链接](https://github.com/maxshuang/Demo/blob/main/algorithm/algorithm/graph/directed_graph/topological.hpp)。
```
void dfs(const Digraph &g)
{
    for (int i = 0; i < g.V() && !has_cycle_; ++i)
    {
        if (!marked_[i])
            dfs_recur(g, i);
    }

    if (!has_cycle_)
        reverse_order();
    else
        post_order_.clear();
}

void dfs_recur(const Digraph &g, int v)
{
    if (has_cycle_)
        return;

    marked_[v] = true;
    path_[v] = true;
    auto pair = g.Adj(v);
    std::for_each(pair.first, pair.second, [&](const Edge &e)
                    {
        if(has_cycle_) return;

        if(!marked_[e.Dest()]) {
            dfs_recur(g, e.Dest());
        }else if(path_[e.Dest()]) {
            has_cycle_=true;
        } });
    // reverse back
    path_[v] = false;
    // postorder traversal
    post_order_.push_back(v);
}

void reverse_order()
{
    std::vector<int> tmp(post_order_.size());
    std::reverse_copy(post_order_.begin(), post_order_.end(), tmp.begin());
    post_order_.swap(tmp);
}
```

Topological Sort 算法本质上一次 DFS， 它的 Time Complexity 为 $O(E+V)$, Space Complexity 为 $O(V)$。

#### 传递闭包(Transitive Closure)

传递闭包是数学上的概念，指是在集合$X$上求包含关系$R$的最小传递关系。[从关系图的角度来说，通途易懂得讲，就是如果原关系图上有 $i$ 到 $j$ 的路径，则其传递闭包的关系图上就应有从 $i$ 到 $j$ 的边](https://zhuanlan.zhihu.com/p/266356742)。

所以本质上传递闭包讨论的是*有向图中顶点对的可达性 reachability*，任意两个顶点可达，则传递闭包的关系图上就有对应的边。

不同于无向图，其传递闭包可以用连通分量或者 Union-Find 算法很好表示，因为边的关系是双向的，有向图的传递闭包问题要复杂一点。举个例子：
```
1 -> 2 -> 3
```
一次从 1 开始的 DFS 就可以确定 {1, 2}, {1, 3}, {2, 3} 是可达，但是却不能确定 {2, 1} 和 {3, 1}是不是可达的。

暴力解法可以解决这个问题，我们分别从所有的顶点开始运行一次 DFS，这样我们就知道 all-pairs reachability。[完整实现版本链接](https://github.com/maxshuang/Demo/blob/main/algorithm/algorithm/graph/directed_graph/transitive_closure.hpp)。
```
TransitiveClosure(const Digraph &g)
{
    for (int i = 0; i < g.V(); ++i)
        dfs_.push_back(DirectDFS(g, i));
}
bool Reachable(int v, int w) const
{
    return dfs_[v].Connected(w);
}
```

也可以用 $V*V$ 矩阵的方式表示 Transitive Closure，本质上和上述实现是一样的，矩阵中的每一行相当于 DirectDFS 类内部的 $marked_[V]$ 标记数组。 

DFS 版本的 Transitive Closure Time Complexity 为 $O(V*(E+V))$, Space Complexity 为 $O(V^{2})$。

以上是 sparse graph 的求解时间复杂度，如果是 dense graph，使用 adjacency matrix $graph[V][V]$ 来表示图，则需要遍历任意两个顶点所有中间顶点的方式确定 reachability，原理上也是暴力遍历，被称为 Floyd Warshall Algorithm。Time Complexity 为 $O(V^{3}))$, Space Complexity 为 $O(V^{2})$。[代码来源](https://www.geeksforgeeks.org/transitive-closure-of-a-graph/)。
```
/* Add all vertices one by one to the
    set of intermediate vertices.
      ---> Before start of a iteration, 
           we have reachability values for
           all pairs of vertices such that
           the reachability values 
           consider only the vertices in 
           set {0, 1, 2, .. k-1} as 
           intermediate vertices.
     ----> After the end of a iteration, 
           vertex no. k is added to the 
            set of intermediate vertices 
            and the set becomes {0, 1, .. k} */
for (k = 0; k < V; k++)
{
    // Pick all vertices as 
    // source one by one
    for (i = 0; i < V; i++)
    {
        // Pick all vertices as 
        // destination for the
        // above picked source
        for (j = 0; j < V; j++)
        {
            // If vertex k is on a path
            // from i to j,
            // then make sure that the value
            // of reach[i][j] is 1
            reach[i][j] = reach[i][j] || 
                (reach[i][k] && reach[k][j]);
        }
    }
}
```

#### 强连通分量(Strongly Connected Component)

类似于无向图中的连通分量，有向图也有强连通分量的概念，因为有向图的边是单向关系，所以有向图的强连通分量一定要成环才能达到互相可达(reachable)的效果。下图展示不同顶点数的强连通分量，单个顶点自己也是一个强连通分量。
![strongly_connected_component](/assets/images/post/algorithm-graph/strongly_connected_component.png)

特别得，我们可以把一个强连通分量看成一个大型顶点，这样整个图就可以描述成正常顶点和大型顶点的 DAG 图。这种大型顶点可能在信息压缩和模型建模上带来一些便利。
![strongly_connected_component2](/assets/images/post/algorithm-graph/strongly_connected_component2.png)

回到求解强连通分量上，从上图中我们可以看到，假如 DFS 遍历起始顶点是 1，则 DFS 无法访问到最近的强连通分量 {0,2,3,4,5}。如果 DFS 遍历起始顶点是 2，则 DFS 可以遍历整个强连通分量 {0,2,3,4,5}，**问题是它无法识别一些非强连通分量顶点，无法区分顶点 1 和顶点 0 的区别**，所以我们需要解决这个问题。

我们首先观察存在强连通分量的图的一些特性：
1. 将强连通分量抽象成大型顶点，则整个图可以描述成一个 DAG;
2. 强连通分量内部顶点是互相可达的，则一次 DFS 至少可以标记一个强连通分量内的所有顶点;
3. 反转强连通分量的边方向，仍然是个强连通分量;

为了解决 DFS 遍历过程中*区分强连通分量顶点和非强连通分量顶点*， 一个重要的想法是: **如果能提前知道顶点间的有向关系**，可以先访问强连通分量的 successor vertex，再访问强连通分量顶点，这样就可以通过访问标记区分不同类型的顶点。

比如上图中，先访问顶点 1，再访问顶点 2，此时从顶点 2 开始的 DFS 会遍历整个强连通分量，但是不会访问到顶点 1, 因为顶点 1 已经被访问过了。也不会访问到顶点 6, 因为整个强连通分量 {0,2,3,4,5} 都对 6 不可达。

*如果能提前知道顶点间的有向关系？*

前面我们提到过，强连通分量本身可以抽象成一个大型顶点，因为它内部顶点是互相可达的，所以在有向关系上内部任意一个顶点都是一样的。整个图其实就是一个 DAG，*而 DAG 的 topological sort 就是顶点间的 precedence contraints，只不过在这个场景下我们需要的是 reverse topological sort*, 因为我们需要从 successor vertex 开始访问，而不是从 precedence vertex 开始访问。

算法思路上非常清晰，实现上遇到的挑战是: *实际图中有环，不是 DAG，无法计算 DAG*。

这里需要清楚一个细节，DAG topological sort 计算本身只是 DFS post-order traversal，任意图都可以通过 DFS 遍历整个图，只是无环 DAG 的 DFS post-order traversal 能表达出 topological sort 的性质。

所以我们仍然可以在含有强连通分量的有向图中运行 DFS post-order traversal，只是它的 reverse post-order 不是 topological sort。比如在上图中运行 DFS post-order traversal，可以出现的 partial post-order 是 {3, 2, 4, 5, 1, 0}, reverse partial post-order 是 {0, 1, 5, 4, 2, 3}，不符合我们需要先访问顶点 1 的要求，原因是在原图上运行 DFS 无法控制访问邻接顶点的次序。

所以为了避免访问到非强连通分量邻接顶点，需要：
1. 反转非强连通分量邻接顶点到强连通分量顶点的有向关系，也就是先把图 reverse;
2. 再在 reverse graph 上运行 DFS post-order traversal。由于强连通分量 {0,2,3,4,5} 在 reverse graph 中仍然是强连通分量，但是强连通分量 {0,2,3,4,5} 不再对顶点 1 可达，从而保证 reverse partial post-order 一定满足 {1, any sequence in{0,2,3,4,5}};
3. 再以 reverse graph's reverse partial post-order 对原图进行 DFS 问题即可搜索到所有的强连通分量;
![reverse_graph](/assets/images/post/algorithm-graph/reverse_graph.png)

上面就是 KosarajuSCC 算法，[完整实现版本链接](https://github.com/maxshuang/Demo/blob/main/algorithm/algorithm/graph/directed_graph/strongly_connected_components.hpp)。
```
StrongComponent(const Digraph& g): marked_(g.V(), false), id_(g.V(), 0), count_(0){ 
    std::vector<int> reverse_order(g.V());
    // 1,2: get reverse postorder traversal result of reverse graph 
    dfs_reverse_graph(g, reverse_order);
    // 3: traverse the origin graph according to the reverse_order
    dfs_graph(g, reverse_order);
}

void dfs_reverse_graph(const Digraph& g, std::vector<int>& reverse_order) {
    Digraph reverse_g=g.Reverse();
    std::vector<int> post_order;
    dfs(reverse_g, post_order);
    std::reverse_copy(post_order.begin(), post_order.end(), reverse_order.begin());
}

void dfs(const Digraph& g, std::vector<int>& post_order) {
    for(int i=0; i<g.V(); ++i) {
        if(!marked_[i]) {
            dfs_recur(g, i, post_order);
        }
    }
}

void dfs_recur(const Digraph& g, int v, std::vector<int>& post_order) {
    marked_[v]=true;
    auto pair=g.Adj(v);
    std::for_each(pair.first, pair.second, [&](const Edge& e){
        if(!marked_[e.Dest()]) {
            dfs_recur(g, e.Dest(), post_order);
        }
    });

    // post order traversal
    post_order.push_back(v);
}

void dfs_graph(const Digraph& g, const std::vector<int>& reverse_order) {
    std::fill(marked_.begin(), marked_.end(), false);
    for(int i: reverse_order) {
        if(!marked_[i]) {
            // a new strongly connected component here
            ++count_;
            dfs_recur(g, i);
        }
    }
}

void dfs_recur(const Digraph& g, int v) {
    marked_[v]=true;
    // all nodes in a strongly connected component will be reachable to each other
    id_[v]=count_;
    
    auto pair=g.Adj(v);
    std::for_each(pair.first, pair.second, [&](const Edge& e){
        if(!marked_[e.Dest()]) {
            dfs_recur(g, e.Dest());
        }
    });
}
```

KosarajuSCC 算法的 Time Complexity 为 $O(E+V)$, 其中 reverse graph Time Complexity 为 $O(E+V)$, 计算 topological sort Time Complexity 为 $O(E+V)$, 原图 DFS Time Complexity 为 $O(E+V)$。Space Complexity 为 $O(V)$。

### 无向图的最小生成树问题(Minimum Spanning Tree)

连通图的生成树是一颗包含所有顶点的树，比如以下黑色边构成的树就是该无向图的一颗生成树。
![minimum_spanning_tree](/assets/images/post/algorithm-graph/minimum_spanning_tree.png)
可以看到生成树有一些非常明显的性质：
1. 图所有顶点通过树的边互相连通;
2. 生成树边数为图顶点数-1(V-1);
3. 任意增加一条边都会在生成树中形成一个环;

无向图的生成树研究的是无向图的连通性问题，一个无向图可以有多个生成树，比如选择上图的 <5-1> 边，去掉 <5-7> 边，就可以得到另外一颗生成树。
[TODO] graph 

最小生成树问题则是研究：*带权无向图中边权值总和最小的生成树*。

这类问题在研究图的连通性领域有非常重要的作用，比如在电路领域研究如何使用最少的连线使得所有组件都能连通，比如在基础设施领域研究如何铺设最少的水泥路使得各个村庄之间互相连通。

该问题已经发展出了非常成熟的 PrimMST 和 KruskalMST 算法，可以分别在 Time Complexity $O(ElogV)$ 和 $O(ElogE)$ 下解决该问题，甚至是更加复杂的 Fredman-Tarjan $O(E+VlogV)$ 算法和 Chazelle nearly $O(E)$ 算法。

有向图的最小生成树问题则被称为最小树型图问题 minimum cost arborescence。

#### 割(Cut property)

PrimMST 和 KruskalMST 算法都基于 Cut property。

Cut property 的概念也很容易理解，形象来讲，想象图是一块蛋糕，*以任何一种方式将蛋糕切分成两个部分(不是切成相同的两半哦)，我们称这种划分方式为一种 Cut*。更加严谨的说法是，将图的顶点划分成两个非空不相交的点集的方式，我们称为一种 Cut。连接两个非空不相交的点集的边称为 crossing edge。

下图中灰色顶点集和白色顶点集的划分就是一种有效的 Cut，红色的边就是 Cut 经过的边，都是 crossing edge。
![cut property](/assets/images/post/algorithm-graph/cut_property.png)

而研究 Cut 和 crossing edge 的原因在于，对于特定的生成树而言，*一个 Cut 的某个 crossing edge 一定是该生成树的边*，因为 crossing edge 决定了两个非空不相交的点集的连通性。

对于最小生成树问题，*一个 Cut 中 weight 最小的 crossing edge 则一定是 MST(Minimum Spanning Tree) 的边*，因为反证法容易证明选择 Cut 中最小 crossing edge 总能获取到边权值总和更小的生成树。

PrimMST 和 KruskalMST 算法的区别在于如何选择最小的 crossing edge。

#### PrimMST 算法

PrimMST 算法选择最小的 crossing edge 的方式非常自然： *从单个顶点的 Cut 开始，每一轮增加一个 tree vertice, 再形成新的 Cut*。
[TODO] graph

单个顶点的 Cut 很简单，起始顶点的所有临接边就是该 Cut 的 crossing edge set，从中选择最小的 crossing edge，它就是*最小生成树的一条有效边，边的另外一个顶点就是另一个 MST vertice*。 这是上一个小节讲的由任意 Cut 性质决定的。

将 new MST vertice 的所有临接边加入到之前的边集合中，就可以形成新的 crossing edge set，然后再选择其中最小的 crossing edge。通过这种每一轮获取一个 new MST vertice，构造 new crossing edge set 的方式，我们可以在获取到 V-1 个 crossing edge 之后就可以确保获得了完整的 MST。

* Lazy PrimMST 算法  
上面这种方法就是我们的 Lazy PrimMST 算法，之所以是 lazy 的是因为会*延迟过滤掉一些不是 crossing edge 的边*。举个例子：
```
graph edge:
<src, dst, weight>
<1, 2, 2.1>
<1, 4, 5.1>
<2, 4, 1.1>

start MST vertice: 1

Round1:
pre MST vertice set: {1}
crossing edge set: {<1, 2, 2.1>, <1, 4, 5.1>}
choose: <1, 2, 2.1>
MST edge set:  {<1, 2, 2.1>}

Round2:
pre MST vertice set: {1, 2}
crossing edge set: {<1, 4, 5.1>, <2, 4, 1.1>}
choose: <2, 4, 1.1>
MST edge set:  {<1, 2, 2.1>, <2, 4, 1.1>}

Round3:
pre MST vertice set: {1, 2, 4}
crossing edge set: {<1, 4, 5.1>}
choose: <1, 4, 5.1>  NOT LEGAL !!! Because <1, 4, 5.1> is not crossing edge anymore, its start vertice and end vertice doesn't belong to different disjoint vertice set.
```

Lazy PrimMST 算法实现如下，[完整实现版本链接](https://github.com/maxshuang/Demo/blob/main/algorithm/algorithm/graph/minimum_spanning_trees/lazy_prim_mst.hpp)。
```
LazyPrimMST(const UndirectedGraph &g) : marked_(g.V(), false), mst_weight_(0.0)
{
    // initialize and build a cut
    visit(g, 0);

    while (!pq_.empty() && !finished())
    {
        auto e = pq_.top();
        pq_.pop();
        // There are some edges which are put into the queue before, but
        // are not crossing edges anymore, just ignore them.
        if (!marked_[e.Dest()])
        {
            mst_.push_back(e);
            mst_weight_ += e.Weight();
            visit(g, e.Dest());
        }
    }
}

void visit(const UndirectedGraph &g, int v)
{
    marked_[v] = true;
    auto pair = g.Adj(v);
    // scan all adjective edges of v
    // ignore the non-crossing edges here if possible
    std::for_each(pair.first, pair.second, [&](const Edge &e)
                    { if(!marked_[e.Dest()])  pq_.push(e); });
}
```

由于 non-crossing edge 的存在，优先队列中的边个数可能达到 $O(E)$ 级别，所以 Lazy PrimMST 算法的时空复杂度为: Time Complexity: $O(ElogE)$, Space Complexity: $O(E)$。

对于 sparse graph 而言，Lazy PrimMST 算法是可用的，实现上也比较简单。对于 dense graph 而言，比如 E 达到了百万或者千万级别，就需要特殊处理下 non-crossing edge，而这就是 PrimMST 算法。

* PrimMST 算法
由于每次都是选择最小的 crossing-edge 加入 MST，则对于优先队列而言，可以只维护从 non-MST vertices 到 MST vertices 中 weight 最小的边即可，只要它才可能会被选择。举个例子：
```
Exist edges:
<1, 2, 5.1>
<1, 3, 2.1>
<3, 2, 1.0>

start MST vertice: 1

Round1:
pre MST vertice set: {1}
crossing edge set: {<1, 2, 5.1>, <1, 3, 2.1>}
choose: <1, 3, 2.1>
MST edge set:  {<1, 3, 2.1>}

Round2:
pre MST vertice set: {1, 3}
we finds that weight(1->3->2) < weight(1->2), so we replace <1, 2, 5.1> with <3, 2, 1.0>
crossing edge set: {<3, 2, 1.0>}
choose: <3, 2, 1.0>
MST edge set:  {<1, 3, 2.1>, <3, 2, 1.0>}
```

实现 PrimMST 算法需要依赖 IndexPriorityQueue，它是 PriorityQueue 的变种，支持给每个 key 关联一个外部 index，以便根据外部 index 直接修改 PriorityQueue key，然后再调整保持堆性质。[完整实现版本链接](https://github.com/maxshuang/Demo/blob/main/algorithm/algorithm/graph/minimum_spanning_trees/prim_mst2.hpp)。
```
PrimMST(const UndirectedGraph &g) : edge_to_(g.V()), marked_(g.V(), false),
                                         weight_to_(g.V(), std::numeric_limits<double>::infinity()), mst_weight_(0.0), pq_(g.V())
{
    // the distance of initial vertice 0 to the mst is 0
    weight_to_[0] = 0.0;
    pq_.Insert(0, 0.0);

    while (!pq_.IsEmpty())
    {
        mst_weight_ += pq_.Top();
        auto i = pq_.Pop();
        // visit the new mst vertices
        visit(g, i);
    }

    make_mst();
}

void visit(const UndirectedGraph &g, int v)
{
    marked_[v] = true;
    auto pair = g.Adj(v);
    std::for_each(pair.first, pair.second, [&](const Edge &e)
                    {
                    // ignore all non-crossing edges
                    if(!marked_[e.Dest()]) {
                        // update non-tree vertice if the new-added vertice has a shorter path to mst
                        if(e.Weight() < weight_to_[e.Dest()]) {
                            weight_to_[e.Dest()]=e.Weight();
                            // edge_to_ will converge to the mst
                            edge_to_[e.Dest()] = e;
                            
                            if(pq_.ContainsIndex(e.Dest())) pq_.Change(e.Dest(), e.Weight());
                            else pq_.Insert(e.Dest(), e.Weight());
                        }
                    } });
}
```

由于优先级队列中只维护 non-tree vertices 到 MST 的最小 edge wegit，所以 Space Complexity 为 $O(V)$, Time Complexity 为 $O(ElogV)$, 因为对优先队列的更新次数是 $O(E)$，每次堆调整是 $O(logV)$。

#### KruskalMST 算法 

KruskalMST 算法在选择最小的 crossing edge 方式上，直接从所有边中持续选择 minimum weight edge，通过检测该边是否属于 crossing edge 来决定它是否属于 MST。该算法的正确性也可以用反证法的方式证明：如果某条边已经是当前所有边中 minimum crossing edge, 则它一定是某个 Cut 中的 minimum crossing edge，它一定属于 MST。

算法实现如下，它需要依赖 Union-Find 算法检测 edge 是否是 crossing edge。[完整实现版本链接](https://github.com/maxshuang/Demo/blob/main/algorithm/algorithm/graph/minimum_spanning_trees/kruskal_mst.hpp)。
```
KruskalMST(const UndirectedGraph& g): pq_(g.Edges().first, g.Edges().second), uf_(g.V()), mst_weight_(0.0) {
    while(!pq_.empty() && mst_edges_.size() < g.V()-1) {
        auto e = pq_.top();
        pq_.pop();
        if(!uf_.Connected(e.Dest(), e.Src())) {
            mst_edges_.push_back(e);
            uf_.Union(e.Dest(), e.Src());
        }
    }
}
```

KruskalMST 算法的 Time Complexity 为 $O(ElogE)$, 其中 Union-Find 能以渐近 $O(1)$ 的方式实现 Connected 和 Union 操作, Space Complexity 为 $O(E)$。

### 图的最短路径问题(Shortest Path Problem)

图的最短路径问题是图的另一类更加有趣且有意义的问题，一般而言它关注的是单源最短路径(Single Source Shortest Path)，从源顶点到图中其他顶点的路径构成一颗最短路径树(SPT, Shortest Paths Tree)。
![shortest paths tree](/assets/images/post/algorithm-graph/spt.png)

对应于现实生活，就是从地点 A 出发，如何选择路线使得到地点 B 经过的路线长度最短。有些时候，图中边的权值不一定就是对应路径长度，它可以是问题相关的指标，比如我们如果关注的是地点 A 到地点 B 之间耗时最短，则边的权值应该是路径耗时，比如 100 米堵车情况下需要 1 小时才能通过，我们可能更加倾向于选择更远的 1000 米只要 10 分钟就能通过的路线。

单源最短路径问题在无向图和有向图上解法是一样的，因为它们关注的都是从源点开始的下一跳。这类问题可以认为已经被很好得解决了，只是需要特殊处理这类问题中存在的一些特别场景，比如存在负权值边或者负权值环。针对不同的场景，可以选择不同的更快的算法，或者使用通用算法。
1. 非负权值图和 DijkstraSP 算法
2. DAG 和 AcyclicSP 算法
3. 一般图和 Bellman-FordSP 算法

在讲解不同算法之前，需要了解 Shortest Path 问题的一个*非常重要的性质*。对于任何一个从源点 s 到目的点 t 的简单最短路径: s -> v1 -> v2 -> ... -> vn -> t，则从源点 s 到该最短路径的任何中间顶点都是简单最短路径，比如：
1. 源点 s 到中间顶点 v2 也是简单最短路径: s -> v1 -> v2
2. 源点 s 到中间顶点 vn 也是简单最短路径: s -> v1 -> v2 -> ... -> vn

这个性质和正负权值无关，它可以用反证法非常容易证明，因为如果源点 s 到该最短路径的任何中间顶点 vi 不是简单最短路径，则说明存在更短的路径到达目的点 t: s -> w1 -> w2 -> ... -> vi -> ... -> vn -> t，这个和假设是冲突的。

> 如果路径是个简单路径，则说明中间不存在负权值环，否则就可以持续通过负权值环构造更短的路径。

这个性质重要点在于：
1. 假如所有的权值都是非负的，则对于一条简单路径而言，它的中间顶点到源顶点的距离是递增的，这是 DijkstraSP 算法可以使用 greedy 算法解决 SP 问题的基础。
2. 对于存在负权值的 DAG，如果已知所有顶点的拓扑序，自然能沿着理论上的最短路径更新所有顶点到源顶点的最短距离。
3. 假如存在正负权值，由于源顶点到所有中间顶点 vi 都是简单最短路径，我们可以在每一轮都获取一个中间顶点，则 V-1 轮后也一定能构造出 s -> v1 -> v2 -> ... -> vn -> t，这是 Bellman-Ford 通用算法求解 SP 问题的基础。

#### 非负权值图和 DijkstraSP 算法

为了简化 SP 问题，我们考虑只存在非负权值边，这在很多场景中也是非常合理的，比如求两点间最短路径，求两点间最短耗时等。

正如上面说的，非负权值边带来的一个性质就是，简单路径上中间顶点到源顶点的距离是递增的，这样我们就可以维护和更新所有其他顶点到源顶点的距离，通过每轮选择其中离源顶点 closest vertice A 作为 SPT vertice。因为没有负权值边，所以不可能存在通过其他顶点到 vertice A 的更短路径了，所以 vertice A 一定是 SPT vertice。

这里涉及到一个非常重要的操作 Relax，它的意思是指每一轮选中 new SPT vertice，就可以通过它的邻接边去更新其他 non-SPT vertice 到源顶点的距离。举个例子：
```
Exist edges:
<1, 2, 5.1>
<1, 3, 2.1>
<3, 2, 1.0>

source vertice: 1
>> original dist 1->2: 5.1
>> new dist 1->3->2: 3.1, through vertice 3, we relax the dist from 1 to 2.  
```

通过这种 greedy 的方式，我们在选择了 V 个 SPT vertice 就可以构造出 SPT， 并回答从源点 s 到其他顶点的最短距离。

* DijkstraSP 算法
DijkstraSP 算法中，我们同样使用 IndexPriorityQueue 维护所有顶点到源顶点的距离，算法流程和 PrimMST 算法基本一样，不同在于 PrimMST 算法在 IndexPriorityQueue 维护的是顶点到 MST 的最小权重。[完整实现版本链接](https://github.com/maxshuang/Demo/blob/main/algorithm/algorithm/graph/shortest_paths/DijkstraSP.hpp)。
```
DijkstraSP(const Digraph &g, int s) : src_(s), edge_to_(g.V()), dist_to_(g.V(), std::numeric_limits<double>::infinity()), pq_(g.V())
{
    dist_to_[s] = 0.0;
    pq_.Insert(s, 0.0);

    while (!pq_.IsEmpty())
    {
        auto i = pq_.Pop();
        visit(g, i);
    }
}
void visit(const Digraph &g, int v)
{
    auto pair = g.Adj(v);
    std::for_each(pair.first, pair.second, [&](const Edge &e)
                    { relax(e); });
}

void relax(const Edge &e)
{
    double dist = dist_to_[e.Src()] + e.Weight();
    if (dist < dist_to_[e.Dest()])
    {
        edge_to_[e.Dest()] = e;
        dist_to_[e.Dest()] = dist;
        if (pq_.ContainsIndex(e.Dest()))
            pq_.Change(e.Dest(), dist);
        else
            pq_.Insert(e.Dest(), dist);
    }
}
```
DijkstraSP 算法 Time Complexity 为 $O(ElogV)$，其中 $O(E)$ 是 relax 的次数，$O(logV)$ 是单次堆调整的时间复杂度。Space Complexity 为 $O(V)$。

* Lazy DijkstraSP 算法
类似 Lazy PrimMST 算法，也可以实现 Lazy DijkstraSP 算法，对于 sparse graph 而言，可以通过不断获取 minimum weight edge 的方式 relax vertice，因为 edge weight 都是非负的，所以最后一定可以收敛并获得 SPT。[完整实现版本链接](https://github.com/maxshuang/Demo/blob/main/algorithm/algorithm/graph/shortest_paths/lazy_DijkstraSP.hpp)。

```
LazyDijkstraSP(const Digraph &g, int s) : src_(s), edge_to_(g.V()), dist_to_(g.V(), std::numeric_limits<double>::infinity())
{
    dist_to_[s] = 0.0;
    visit(g, s);

    while (!pq_.empty())
    {
        auto e = pq_.top();
        pq_.pop();
        relax(g, e);
    }
}

void visit(const Digraph &g, int v)
{
    auto pair = g.Adj(v);
    std::for_each(pair.first, pair.second, [&](const Edge &e)
                    { pq_.push(e); });
}

void relax(const Digraph &g, const Edge &e)
{
    double dist = dist_to_[e.Src()] + e.Weight();
    if (dist < dist_to_[e.Dest()])
    {
        dist_to_[e.Dest()] = dist;
        edge_to_[e.Dest()] = e;
        visit(g, e.Dest());
    }
}
```
Lazy DijkstraSP 算法 Time Complexity 为 $O(ElogE)$，其中 $O(E)$ 是 relax 的次数，$O(logE)$ 是单次堆调整的时间复杂度。Space Complexity 为 $O(E)$。

#### 有向无环图(DAG)和 AcyclicSP 算法

当图中存在负权值时，DijkstraSP 算法就无法使用了，因为最短路径中中间顶点到源顶点的距离不是递增的，所以使用 greedy algorithm 在每轮选择离源顶点 closest vertice 作为 SPT vertice 就无法提供理论上的保证。

这一小节讨论存在负权值边的 DAG 中 Shortest Path 求解。DAG 一个非常重要的特性就是 DAG 可以获取到 topological order，而 topological order 反映的是 vertice 之间入边的依赖关系。按照 topological order 对 vertice 进行 relax, 我们可以依赖链式推导关系知道，current vertice 离源点所有可能的 dist 都被计算了一次，并且 all precedence vertices 也被计算了一次, all pre-precedence vertices 也被计算了一次...。
举个例子：
```
Exist edges:
<1, 2, 1>
<2, 3, 1>
<3, 4, 1>
<1, 4, 5>

For this graph, its topological order is {1, 2, 3, 4}, 
so we can get the minimum dist from 1 to 4 is 3, not 5, 
because we know that after relax vertice3, 
no other vertice can relax the dist from 1 to 4.

If we relax the vertice at {1, 2, 4, 3}, 
then we may think that minimum dist from 1 to 4 is 5, 
because we have no idea then when we can get the minimum dist at a casual relax sequence.
```
基于 DAG topological order 实现如下，[完整实现版本链接](https://github.com/maxshuang/Demo/blob/main/algorithm/algorithm/graph/shortest_paths/acyclicSP.hpp)。
```
AcyclicSP(const Digraph &g, int s) : src_(s), edge_to_(g.V()), dist_to_(g.V(), std::numeric_limits<double>::infinity())
{
    dist_to_[s] = 0.0;
    // calculate Topological sort
    Topological tp(g);
    if (!tp.IsDAG())
        return;
    relax_all(g, tp.Order());
}

void relax_all(const Digraph &g, const std::vector<int> &vs)
{
    int st = 0;
    // skip precedence vertice to src_
    // they are all unreachable
    for (; st < (int)vs.size(); ++st)
    {
        if (vs[st] == src_)
            break;
    }
    for (; st < (int)vs.size(); ++st)
    {
        auto pair = g.Adj(vs[st]);
        std::for_each(pair.first, pair.second, [&](const Edge &e)
                        { relax(e); });
    }
}

void relax(const Edge &e)
{
    double dist = dist_to_[e.Src()] + e.Weight();
    if (dist < dist_to_[e.Dest()])
    {
        dist_to_[e.Dest()] = dist;
        edge_to_[e.Dest()] = e;
    }
}
```

AcyclicSP 算法 Time Complexity 为 $O(E+V)$, 其中计算 DAG topological order Time Complexity 为 $O(E+V)$, 而基于 topological order 进行 relax vertice Time Complexity 为 $O(E+V)$。
AcyclicSP 算法 Space Complexity 为 $O(V)$.

#### 一般图和 Bellman-FordSP 算法

以上讨论的都是一些特殊图的最短路径解法，其中 DijkstraSP 算法依赖非负权值，AcyclicSP 算法依赖 DAG，对于存在正负权值且有正负环存在的图，就需要更加通用的 Bellman-Ford 算法。

对于求解最短路径问题，正环的存在不影响结果，所以我们不考虑正环。从源顶点开始如果经过负环，则认为负环上的所有顶点都无法构造最短路径，因为沿着负环总可以构造出更短的路径。

对于一般图而言，我们定义最短路径为：
1. 如果从源顶点不可达，则不存在最短路径，距离为正无穷大 infinity；
2. 如果从源顶点可达，但是为负环上的顶点，则不存在最短路径，距离为负无穷大 -infinity；
3. 如果从源顶点可达，且不是负环上的顶点，则存在简单最短路径；

Bellman-Ford 算法基于这样的直觉：
*对于不存在源顶点可达负环的有向图，在每轮中，以任意顺序对所有的 edge 进行 relax，至少可以获得 1 个 SPT vertice。进行 V-1 轮后一定可以获得 SPT。*

这是一个第一眼看起来**不太合理**的直觉，所以我们看下一个任意的最短路径，假设是: 
s -> v1 -> v2 -> v3 -> t
对于 Bellman-Ford 算法而言，如果在 round i 找到了 s -> v1 -> v2，则在 round i+1 以任意顺序 relax edge 都一定可以找到 v3。

这个结论是成立的，因为在 round i+1 以任意顺序 relax edge，s -> v1 -> v2 不会有任何变更，因为他们已经都是最短路径了，此时以任意顺序 relax v2 的邻接边时，dist_to[v3] 会被更新成 dist_to[v2] + weight(v2, v3), 而这就是 dist_to[v3] 的最小值，因为我们假设了存在 s -> v1 -> v2 -> v3 -> t 这样一条最短路径。

> 即使存在 v2 -> v4 -> v3 这样的路径更短，也可以在 round i+1 找到 v4, 在 round i+2 找到 v3, 当然这和我们一开始的假设已经不一致了，只是为了说明他们的原理都是一样的。

所以 Bellman-Ford 算法可以认为是一种暴力解法，理论上证明了即使以任意顺序 relax edge 也可以在 V-1 轮后获取 SPT。如果存在源顶点可达的负环，则无法在 V-1 轮后结束，所以可以在 V-1 轮后就检测 SPT 是否存在环，从而检测原始图中负环的存在。

* Lazy Bellman-FordSP 算法
上面描述的是 Lazy Bellman-FordSP 算法，直接进行 V-1 轮 relax，[完整实现版本链接](https://github.com/maxshuang/Demo/blob/main/algorithm/algorithm/graph/shortest_paths/lazy_Bellman_Ford.hpp)。

```
LazyBellmanFordSP(const Digraph &g, int s) : src_(s), edge_to_(g.V()), dist_to_(g.V(), std::numeric_limits<double>::infinity())
{
    dist_to_[s] = 0.0;
    // for any digraph without negative cycles, it will converge to spt after V round
    for (int r = 0; r < g.V(); ++r)
    {
        for (int v = 0; v < g.V(); ++v)
        {
            auto pair = g.Adj(v);
            std::for_each(pair.first, pair.second, [&](const Edge &e)
                            {
                    if(e.Weight()+dist_to_[e.Src()] < dist_to_[e.Dest()]) {
                    edge_to_[e.Dest()]=e;
                    dist_to_[e.Dest()]=e.Weight()+dist_to_[e.Src()];
                    } });
        }
    }

    findNegativeCycle();
}

void findNegativeCycle()
{
    std::vector<bool> marked(edge_to_.size(), false);
    int negative_cycle_vertice = -1;
    for (int i = 0; i < (int)edge_to_.size() && !has_negative_cycle_; ++i)
    {
        if (marked[i])
            continue;
        int w = i;
        while (w != -1)
        {
            marked[w] = true;
            w = edge_to_[w].Src();
            if (w == i)
            {
                has_negative_cycle_ = true;
                negative_cycle_vertice = i;
                break;
            }
        }
    }

    if (has_negative_cycle_)
    {
        // get negative cycle
        int w = negative_cycle_vertice;
        while (edge_to_[w].Src() != negative_cycle_vertice)
        {
            negative_cycle_.push_back(edge_to_[w]);
            w = edge_to_[w].Src();
        }
        negative_cycle_.push_back(edge_to_[w]);
    }
}
```

Lazy Bellman-FordSP 算法的 Time Complexity 为 $O(EV)$，其中需要进行 $O(V)$ 轮，每一轮需要 relax $O(E)$ 条边。空间复杂度为 $O(V)$。


* Bellman-FordSP 算法
更高效率的 Bellman-FordSP 算法只会使用上一轮被 update 过的 vertice 进行 relax，因为只有他们的邻接边才会导致其他顶点到源顶点的距离被更新。

具体实现如下，[完整实现版本链接](https://github.com/maxshuang/Demo/blob/main/algorithm/algorithm/graph/shortest_paths/Bellman_Ford.hpp)。

```
BellmanFordSP(const Digraph &g, int s) : src_(s), edge_to_(g.V()), dist_to_(g.V(), std::numeric_limits<double>::infinity()), on_que_(g.V(), false)
{
    dist_to_[s] = 0.0;
    on_que_[s] = true;
    // start from the source vertice
    que_.push(s);
    int round=0;
    while (!que_.empty())
    {
        int v = que_.front();
        que_.pop();
        on_que_[v] = false;
        auto pair = g.Adj(v);
        std::for_each(pair.first, pair.second, [&](const Edge &e)
                        {
                    if(e.Weight()+dist_to_[e.Src()] < dist_to_[e.Dest()]) {
                    edge_to_[e.Dest()]=e;
                    dist_to_[e.Dest()]=e.Weight()+dist_to_[e.Src()];
                    // if vertice is not on queue, push it for next round
                    if(!on_que_[e.Dest()]) {
                        que_.push(e.Dest());
                        on_que_[e.Dest()]=true;
                    }
                    } });

        
        // a digraph without negative cycles will converge at V round
        if(++round%g.V()==0) {
            findNegativeCycle();
            break;
        }
    }
}
```

Bellman-FordSP 算法一般情况下 Time Complexity 可以达到 $O(E+V)$，其中大概平均每一轮只有 1 个 vertice 会被 relax，总共有 $O(E)$ 条边被 relax。Worst Time Complexity 和 Lazy Bellman-FordSP 算法一样都是 $O(EV)$。空间复杂度为 $O(V)$。