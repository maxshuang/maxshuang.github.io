---
layout: post
title: Linux Kernel-Algorithm-Graph(WIP)
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

## 图
图结构是现实世界的一个重要抽象，表达的是多对多的关系，广泛存在于多种场景中，比如：
* 地图：非常直接的图关系，地点是节点，道路是路径，在抽象时可能使用其他属性，比如汽车行驶花费的时长，作为节点间的连接路径。
* 人际关系和社交网络：每个人有多个好友，好友又有好友，可能存在环状的相识链，好友关系一般是无向的关系，社交网络中关注关系则是有向的关系。
* 网站超链接：网站作为经典的分布式应用，网站内部通过维护其他网站超链接的方式维护网站间关系。
* 商品调度/贸易关系/软件设计：这些场景都涉及了多个模块之间的交互，自然存在节点和连接的图抽象

一个经典的图抽象可以表达为如下：
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
5. 生成树(spanning tree)：一个图中连接所有节点的树称为生成树，比如下图中阴影部分就是各个联通分量的生成树：
![spanning-tree-forest](/assets/images/post/algorithm-graph/spanning-tree-forest.png)
6. 森林(forest)和生成树森林(spanning forest): 多个树组成森林，多个生成树组成生成树森林。

## 图的表示和操作
图在大的分类上分成有向图和无向图。

在下面的表示中，为了结构清晰，参考[算法4]()我们会将图的表示分成图行为和图实现，并且分离出图搜索等相关算法。其中：
* 图行为：提供一个图的基本操作，比如节点个数，边个数，特点节点的节点集等。
* 图实现：具体实现图的方法，比如是使用数组还是链表存储节点和边集合。
* 图操作：包括图搜索等算法，每个操作可以独立开来，只和图行为交互，和图表示没有关系。

在描述图和图算法过程中，我们使用面向对象的语言 C++ or Java，面向对象的设计方法可以有效实现隐藏相关实现，并且通过 public 接口定义可以明确各组件的交互边界。

### 无向图
无向图中所有节点间连接都是无向的。图行为定义如下：
```
/**
 * @struct Edge
 * @brief Represents an edge in the graph.
 */
struct Edge {
    Edge(int s, int d) : src(s), dest(d) {}
    int Src() const { return this->src; }
    int Dest() const { return this->dest; }
};

/**
 * @struct UndirectedGraphWithoutWeight
 * @brief Represents an undirected graph without edge weights.
 */
class UndirectedGraphWithoutWeight {
//
// behavior
//
public:
    UndirectedGraphWithoutWeight(int V);
    ~UndirectedGraphWithoutWeight();
    void AddEdge(int v, int w);
    int V() const;
    int E() const;
    // [TODO] make it implementation-free
    std::forward_list<Edge> Adj(int v) const;
    int Degree(int v) const;
    int MaxDegree() const;
    float AvgDegree() const;
    int NumberOfSelfLoops() const;
    void Show() const;
};
```
我们定义了一个图提供的基本行为，其中比较核心的是获取特定节点的临接节点列表`std::forward_list<Edge> Adj(int v) const;`，这里有 3 个小问题需要注意：
1. 定义 public 接口是为了从设计上定义并限定这个 class 可以提供什么样的交互能力，需要能做到实现无关。
2. 按照设计原则，Adj 不应该返回 std::forward_list 对象，因为这个是 C++ 标准库中定义的单向链表，已经是一种实现绑定了。这个主要是 C++ 标准库设计中没有类似 Java Iterator interface 这种只返回一个定义行为的设计。
```
class Iterator {
public:
    Iterator Next(){}
    Iterator End(){ return }
private：
    std::forward_list<Edge>::const_iterator cur;
    std::forward_list<Edge>::const_iterator end;
}

```


