---
layout: post
title: Linux Kernel-Algorithm-Graph(Basic)
subtitle: picture from https://www.pexels.com/search/wild%20animals/
author: maxshuang
categories: Algorithm
banner:
  image: /assets/images/post/linux-kernel-process-scheduler/pexels-rajitha-fernando-1557917.jpg
  opacity: 0.618
  background: "#000"
  height: "70vh"
  min_height: "38vh"
  heading_style: "font-size: 3.00em; font-weight: bold; text-decoration: underline"
  subheading_style: "color: gold"
tags: Algorithm Graph
---

> NOTE: 内容来自于 [算法4](https://algs4.cs.princeton.edu/home/), 主要记录个人对其中算法的理解。有兴趣或者需要实现源码可以直接阅读 [online website graph](https://algs4.cs.princeton.edu/40graphs/). Enjoy it:)

## 图
图结构是现实世界的一个重要抽象，表达的是节点间关联关系，广泛存在于多种场景中，比如：
* 地图：非常直接的图关系，地点是节点，道路是路径，在抽象时可能使用其他属性，比如汽车行驶花费的时长，作为节点间的连接路径。
* 人际关系和社交网络：每个人有多个好友，好友又有好友，可能存在环状的相识链，好友关系一般是无向的关系，社交网络中关注关系则是有向的关系。
* 网站超链接：网站作为经典的分布式应用，网站内部通过维护其他网站超链接的方式组成一个超大规模图。
* 商品调度/贸易关系/软件设计：这些场景都涉及了多个模块之间的交互，自然存在节点和连接的图抽象。

经典的图抽象可以分成无向图和有向图：
![undirected-graph](/assets/images/post/algorithm-graph/classic-graph.png)
![directed-graph](/assets/images/post/algorithm-graph/classic-directed-graph.png)

## 图的性质
图由节点和边构成，存在以下性质和相关概念：  
![attributes](/assets/images/post/algorithm-graph/anatomy-of-a-graph.png)
1. 节点度(degree)/入度(in-degree)/出度(out-degree)：直接连接到该节点的边个数称为节点的度，在有向图中，度被进一步区分成入度和出度。
2. 路径(path)/路径长度(length of path)：边连接的一系列节点称为路径，路径长度为边的个数。
3. 联通(connected)/联通分量(connected component): 当每个节点对其他节点都存在至少一条路径时我们称图是联通的；联通分量表示图中互相联通的节点集合；一个图中可能含有多个联通分量。
4. 无环图(acyclic graph): 一个图中没有环则称为无环图，是一颗树。
![acyclic-graph](/assets/images/post/algorithm-graph/acyclic-graph.png)
5. 生成树(spanning tree)：一个图中连接所有节点的树称为生成树，比如下图中阴影部分就是各个联通分量的生成树。生成树在研究*图连通性*和*最小连通问题*等问题领域都有重要作用。
![spanning-tree-forest](/assets/images/post/algorithm-graph/spanning-tree-forest.png)
6. 森林(forest)和生成树森林(spanning forest): 多个树组成森林，多个生成树组成生成树森林。

## 图的表示和操作
图在大的分类上分成无向图和有向图，无向图的边是双向的，右向图的边是单向的。

在下面的表示中，为了结构清晰，参考[算法4]()我们会将图的表示分成图行为和图实现，并且分离出图搜索等相关算法。其中：
* 图行为：提供一个图的基本操作，比如节点个数，边个数，特点节点的节点集等。
* 图实现：具体实现图的方法，比如是使用数组还是链表存储节点和边集合。
* 图操作：包括图搜索等算法，每个操作可以独立开来，只和图行为交互，和图表示没有关系。

在描述图和图算法过程中，我们使用面向对象的语言 C++，面向对象的设计方法可以有效实现隐藏相关实现，并且通过 public 接口定义可以明确各组件的交互边界。

### 图的操作
无向图中所有节点间连接都是无向的。图行为定义如下：
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

private:
    //
    //  implementation...
    //
};

/**
 * @struct UndirectedGraph
 * @brief Represents an undirected graph without edge weights.
 */
class UndirectedGraph
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
private:
    //
    //  implementation
    //
};
```


### 图的深度遍历(DFS)和宽度遍历(BFS)



### 无向图

#### 环检测

#### 连通分量(Connected Component)


### 有向图

#### 环检测和 DAG(Directed Acyclic Graph)

#### DAG 和拓扑排序(Topological Sort)

#### 强连通分量(Strong Connected Component)

#### 传递闭包(Transitive Closure)


### 无向图的最小生成树问题(Minimum Spanning Tree)
连通图的生成树是一颗包含所有节点的树，比如以下黑色边构成的树就是该无向图的一颗生成树。
![alt text](image.png)
可以看到生成树有一些非常明显的性质：
1. 图所有节点通过树的边互相连通;
2. 生成树边数为图节点数-1(V-1);
3. 任意增加一条边都会在生成树中形成一个环;

无向图的生成树研究的是无向图的连通性问题，一个无向图可以有多个生成树，比如选择上图的 <5-1> 边，去掉 <5-7> 边，就可以得到另外一颗生成树。
[TODO] graph 

最小生成树问题则是研究：*带权无向图中边权值总和最小的生成树*。

这类问题在研究图的连通性领域有非常重要的作用，比如在电路领域研究如何使用最少的连线使得所有组件都能连通，比如在基础设施领域研究如何铺设最少的水泥路使得各个村庄之间互相连通。

该问题已经发展出了非常成熟的 PrimMST 和 KruskalMST 算法，可以分别在 Time Complexity $O(ElogV)$ 和 $O(ElogE)$ 下解决该问题，甚至是更加复杂的 Fredman-Tarjan $O(E+VlogV)$ 算法和 Chazelle nearly $O(E)$ 算法。

有向图的最小生成树问题则被称为最小树型图问题 minimum cost arborescence。

#### 割(Cut property)
PrimMST 和 KruskalMST 算法都基于 Cut property。

Cut property 的概念也很容易理解，形象来讲，想象图是一块蛋糕，*以任何一种方式将蛋糕切分成两个部分(不是切成相同的两半哦)，我们称这种划分方式为一种 Cut*。更加严谨的说法是，将图的节点划分成两个非空不相交的点集的方式，我们称为一种 Cut。连接两个非空不相交的点集的边称为 crossing edge。

下图中灰色节点集和白色节点集的划分就是一种有效的 Cut，红色的边就是 Cut 经过的边，都是 crossing edge。
![alt text](image-1.png)

而研究 Cut 和 crossing edge 的原因在于，对于特定的生成树而言，*一个 Cut 的某个 crossing edge 一定是该生成树的边*，因为 crossing edge 决定了两个非空不相交的点集的连通性。

对于最小生成树问题，*一个 Cut 中 weight 最小的 crossing edge 则一定是 MST(Minimum Spanning Tree) 的边*，因为反证法容易证明选择 Cut 中最小 crossing edge 总能获取到边权值总和更小的生成树。

PrimMST 和 KruskalMST 算法的区别在于如何选择最小的 crossing edge。

#### PrimMST 算法
PrimMST 算法选择最小的 crossing edge 的方式非常自然： *从单个节点的 Cut 开始，每一轮增加一个 tree vertice, 再形成新的 Cut*。
[TODO] graph

单个节点的 Cut 很简单，起始节点的所有临接边就是该 Cut 的 crossing edge set，从中选择最小的 crossing edge，它就是*最小生成树的一条有效边，边的另外一个顶点就是另一个 MST vertice*。 这是上一个小节讲的由任意 Cut 性质决定的。

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
图的最短路径问题是图的另一类更加有趣且有意义的问题，一般而言它关注的是单源最短路径(Single Source Shortest Path)，从源节点到图中其他节点的路径构成一颗最短路径树(SPT, Shortest Paths Tree)。
![alt text](image-2.png)

对应于现实生活，就是从地点 A 出发，如何选择路线使得到地点 B 经过的路线长度最短。有些时候，图中边的权值不一定就是对应路径长度，它可以是问题相关的指标，比如我们如果关注的是地点 A 到地点 B 之间耗时最短，则边的权值应该是路径耗时，比如 100 米堵车情况下需要 1 小时才能通过，我们可能更加倾向于选择更远的 1000 米只要 10 分钟就能通过的路线。

单源最短路径问题在无向图和有向图上解法是一样的，因为它们关注的都是从源点开始的下一跳。这类问题可以认为已经被很好得解决了，只是需要特殊处理这类问题中存在的一些特别场景，比如存在负权值边或者负权值环。针对不同的场景，可以选择不同的更快的算法，或者使用通用算法。
1. 非负权值图和 DijkstraSP 算法
2. DAG 和 AcyclicSP 算法
3. 一般图和 Bellman-FordSP 算法

在讲解不同算法之前，需要了解 Shortest Path 问题的一个*非常重要的性质*。对于任何一个从源点 s 到目的点 t 的简单最短路径: s -> v1 -> v2 -> ... -> vn -> t，则从源点 s 到该最短路径的任何中间节点都是简单最短路径，比如：
1. 源点 s 到中间节点 v2 也是简单最短路径: s -> v1 -> v2
2. 源点 s 到中间节点 vn 也是简单最短路径: s -> v1 -> v2 -> ... -> vn

这个性质和正负权值无关，它可以用反证法非常容易证明，因为如果源点 s 到该最短路径的任何中间节点 vi 不是简单最短路径，则说明存在更短的路径到达目的点 t: s -> w1 -> w2 -> ... -> vi -> ... -> vn -> t，这个和假设是冲突的。

> 如果路径是个简单路径，则说明中间不存在负权值环，否则就可以持续通过负权值环构造更短的路径。

这个性质重要点在于：
1. 假如所有的权值都是非负的，则对于一条简单路径而言，它的中间节点到源节点的距离是递增的，这是 DijkstraSP 算法可以使用 greedy 算法解决 SP 问题的基础。
2. 对于存在负权值的 DAG，如果已知所有顶点的拓扑序，自然能沿着理论上的最短路径更新所有顶点到源顶点的最短距离。
3. 假如存在正负权值，由于源节点到所有中间节点 vi 都是简单最短路径，我们可以在每一轮都获取一个中间节点，则 V-1 轮后也一定能构造出 s -> v1 -> v2 -> ... -> vn -> t，这是 Bellman-Ford 通用算法求解 SP 问题的基础。

#### 非负权值图和 DijkstraSP 算法
为了简化 SP 问题，我们考虑只存在非负权值边，这在很多场景中也是非常合理的，比如求两点间最短路径，求两点间最短耗时等。

正如上面说的，非负权值边带来的一个性质就是，简单路径上中间节点到源节点的距离是递增的，这样我们就可以维护和更新所有其他节点到源节点的距离，通过每轮选择其中离源节点 closest vertice A 作为 SPT vertice。因为没有负权值边，所以不可能存在通过其他节点到 vertice A 的更短路径了，所以 vertice A 一定是 SPT vertice。

这里涉及到一个非常重要的操作 Relax，它的意思是指每一轮选中 new SPT vertice，就可以通过它的邻接边去更新其他 non-SPT vertice 到源节点的距离。举个例子：
```
Exist edges:
<1, 2, 5.1>
<1, 3, 2.1>
<3, 2, 1.0>

source vertice: 1
>> original dist 1->2: 5.1
>> new dist 1->3->2: 3.1, through vertice 3, we relax the dist from 1 to 2.  
```

通过这种 greedy 的方式，我们在选择了 V 个 SPT vertice 就可以构造出 SPT， 并回答从源点 s 到其他节点的最短距离。

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
当图中存在负权值时，DijkstraSP 算法就无法使用了，因为最短路径中中间顶点到源顶点的距离不是递增的，所以使用 greedy algorithm 在每轮选择离源节点 closest vertice 作为 SPT vertice 就无法提供理论上的保证。

这一小节讨论存在负权值边的 DAG 中 Shortest Path 求解。DAG 一个非常重要的特性就是 DAG 可以获取到 topological order，而 topological order 反映的是 vertice 之间入边的依赖关系。按照 topological order 对 vertice 进行 relax, 我们可以依赖链状推导关系知道，current vertice 离源点所有可能的 dist 都被计算了一次，并且 all precedence vertices 也被计算了一次, all pre-precedence vertices 也被计算了一次...。
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

对于求解最短路径问题，正环的存在不影响结果，所以我们不考虑正环。从源节点开始如果经过负环，则认为负环上的所有顶点都无法构造最短路径，因为不存在。所以对于一般图而言，我们定义最短路径为：
1. 如果从源顶点不可达，则不存在最短路径，距离为正无穷大 infinity；
2. 如果从源顶点可达，但是为负环上的顶点，则不存在最短路径，距离为负无穷大 -infinity；
3. 如果从源顶点可达，且不是负环上的顶点，则存在简单最短路径；

Bellman-Ford 算法基于这样的直觉：
*对于不存在源顶点可达负环的有向图，在每轮中，以任意顺序对所有的 edge 进行 relax，至少可以获得 1 个 SPT vertice。进行 V-1 轮后一定可以获得 SPT。*

这是一个第一眼看起来**不太合理**的直觉，所以我们看下一个任意的最短路径，假设是: 
s -> v1 -> v2 -> v3 -> t
对于 Bellman-Ford 算法而言，如果在 round i 找到了 s -> v1 -> v2，则在 round i+1 以任意顺序 relax edge 都一定可以找到 v3。
这个结论是成立的，因为在 round i+1 以任意顺序 relax edge，s -> v1 -> v2 不会有任何变更，因为他们已经都是最短路径了，此时以任意顺序 relax v2 的邻接边时，dist_to[v3] 会被更新成 dist_to[v2] + weight(v2, v3), 而这就是 dist_to[v3] 的最小值，因为我们假设了存在 s -> v1 -> v2 -> v3 -> t 这样一条最短路径。即使存在 v2 -> v4 -> v3 这样的路径更短，也可以在 round i+1 找到 v4, 在 round i+2 找到 v3, 当然这和我们一开始的假设已经不一致了，只是为了说明他们的原理都是一样的。

所以 Bellman-Ford 算法可以认为是一种暴力解法，理论上证明了即使以任意顺序 relax edge 也可以在 V-1 轮后获取 SPT。如果存在源顶点可达的负环，则无法在 V-1 轮后结束，所以可以在 V-1 轮后就检测 SPT 是否存在环，从而检测原始图中负环的存在。

* Lazy Bellman-FordSP 算法
上面描述的是Lazy Bellman-FordSP 算法，直接进行 V-1 轮 relax，[完整实现版本链接](https://github.com/maxshuang/Demo/blob/main/algorithm/algorithm/graph/shortest_paths/lazy_Bellman_Ford.hpp)。

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

* Bellman-FordSP 算法
更好效率的 Bellman-FordSP 算法只会使用上一轮被 update 过的 vertice 进行 relax，因为只有他们的邻接边才会导致其他节点到源顶点的距离被更新。

具体实现如下，[完整实现版本链接](https://github.com/maxshuang/Demo/blob/main/algorithm/algorithm/graph/shortest_paths/Bellman_Ford.hpp)。

```
BellmanFordSP(const Digraph &g, int s) : src_(s), edge_to_(g.V()), dist_to_(g.V(), std::numeric_limits<double>::infinity()), on_que_(g.V(), false)
{
    dist_to_[s] = 0.0;
    on_que_[s] = true;
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