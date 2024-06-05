
# Lowest Common Ancestor

LCA 的一些基础性质：
1. $d(u, v)=h(u)+h(v)-2*h(LCA(u, v))$ for edge
2. $d(u, v)=h(u)+h(v)-h(LCA(u, v))-h(Fa(LCA(u, v)))$ for edge

如何高效得获取 Fa(LCA) ?


## DFS 算法
求 LCA 有个 O(N) 算法，本质上是 DFS
预处理 O(N), 每次查询都是 O(N)

栈实现 LCA：
```
Node* dfs(root, p, q) {
    if(!root || root==p || root==q) return root;

    Node* p=dfs(root->left, p, q);
    Node* q=dfs(root->right, p, q);

    if(!p) return q;
    if(!q) return p;

    return root; // lca node
}
```

post-order dfs 本身就是存储一跳父节点的实现方案。

队列实现：
其实就是通过 dfs 获取 p 和 q 的所有父节点，然后从 root 节点开始对比直到找到第一个共同节点就是 lca：
```
bool findAncestor(root, p, array) {
    if(!root || root==p) return true

    array.push_back(root);

    // dfs find the path from root to p
    if(findAncestor(p->left, p, array) || 
       findAncestor(p->right, p, array)) {
        return true; // find the p
    } 

    // revert back
    array.pop_back(root);
    return false;
} 

p_array=[];
q_array=[];

findAncestor(root, p, p_array);
findAncestor(root, q, q_array);

for(pn: p_array; qn: q_array) {
    if(pn==qn) return pn;  // lca here
}

```

## Binary Lifting
空间换时间，通过存储多级父节点进行快速跳转，而不是一个一个查找。

```
struct TreeNode {
    int val;
    TreeNode *left;
    TreeNode *right;
    int depth;
};

const int MAXLEVEL = 10, MAX_NODE=1000;

// Father_X_I is the 2^i fathers of a node
typedef std::vector<TreeNode*> Father_X_I;
std::map<TreeNode*, Father_X_I> fathers;

// table for lg2
std::vector<int> lg2(MAX_NODE, 0);

void initLG2() {
    for(size_t i=2; i<lg2.size(); ++i) {
        lg2[i]=lg2[i/2]+1;
    }
}

void dfs(TreeNode* node, TreeNode* fa) {
    if(!node) return;

    // update depth
    if(fa) node->depth=fa->depth+1;

    // update father relationship
    if(fathers.count(node)==0) {
        fathers.insert(std::make_pair(node, Father_X_I(MAXLEVEL, nullptr)));
    }
    fathers[node][0]=fa;

    // all fathers have been set before
    for(int i=1; i<=lg2[node->depth]; ++i) {
        fathers[node][i]=fathers[fathers[node][i-1]][i-1];   // 2^i=2^(i-1)+2^(i-1)
    }

    dfs(node->left, node);
    dfs(node->right, node);
}

// O(NlogN) for preprocess, O(N) for DFS, O(logN) for building father relationship for each node
void init(TreeNode* root) {
    initLG2();
    // preprocess, get node's depth and father relationship
    dfs(root, nullptr);
}

// O(logN) for each search
TreeNode* findLCA(TreeNode *root, TreeNode *p, TreeNode *q) {
    // move p and q to the same level
    // let p be the lower one
    if(p->depth<q->depth) {
        std::swap(p, q);
    }

    // move p to the same depth with q
    while(p->depth != q->depth) {
        int diff = lg2[p->depth-q->depth];
        p=fathers[p][diff];
    }

    // p or q is the lca
    if(p==q) return p;

    // p and q jumps to the lca
    for(int k=lg2[p->depth-root->depth]; k>=0; --k) {
        // nodes precedent to lca will all be the same
        if(fathers[p][k]!=fathers[q][k]) {
            p=fathers[p][k];
            q=fathers[q][k];
        }
    }

    return fathers[p][0]; // father is the lca
}
```

链状数组实现方式：

```
#include <bits/stdc++.h>
using namespace std;
const int N=500005,M=500005; //N存储节点总数,M存储边的总数
struct edge{
    int to, next;
}e[M*2]; //链状数组
int head[N],deep[N],fa[N][22],lg2[N];//head[]为每个节点的起始边，deep[]记录深度
//fa[i][j]表示i的2^j级祖先
//lg2数组记录log2(i), 用来记录在特定深度下最多需要记录多少个父节点
int tot=0;//tot统计边数 
bool vis[N];//节点访问标记 

// 本质上就是邻接边的形式表示树，只不过用链状数组存储了所有链表
void add(int x,int y)
{
        // 从 1 开始
    e[++tot].to=y;//存储节点
    e[tot].next=head[x];//链表
    head[x]=tot;//标记节点位置
}

void dfs(int x,int y=0)
{
    if(vis[x]) return;//已经访问过，则已经得到deep和fa数组，返回 
    vis[x] = true;
    deep[x]=deep[y]+1;//x是y的儿子节点,所以要+1
    fa[x][0]=y;//fa[x][0]表示x的父亲节点,而y是x的父亲节点.
        // 因为到这里说明所有的父节点信息都被初始化过了
    for(int i=1; i<=lg2[deep[x]]; i++) //因为x的2^i级祖先最多是根，则2^i<=deep[x],i<=lg2[deep[x]] 
        fa[x][i]=fa[fa[x][i-1]][i-1];//状态转移 2^i=2^(i-1)+2^(i-1)
         // 链状数组访问子结点方式
         for(int i=head[x]; i; i=e[i].next)
        dfs(e[i].to, x);//访问儿子节点,并且标记自己是父亲节点
}

int LCA(int x,int y)
{
    if(deep[x]<deep[y])//不妨让x节点是在y下方的节点
        swap(x,y);//交换,维持性质
    while(deep[x] != deep[y])//当我们还没有使得节点同样深度
        x=fa[x][lg2[deep[x]-deep[y]]];//往上面跳跃,deep[x]-deep[y]是高度差. // lg2() 是最大的 1 对应的值
    if(x==y)//发现Lca(x,y)=y
        return x;//返回吧,找到了...
         // depth 可以分解成 2 进制，所以从最大的步子跳再逐步变小肯定能跳到
         // 4(2^2) => (2+1)+1,  3 => (2) + 1
         for(int k=lg2[deep[x]]; k>=0; k--) //如何在x,y不相遇的情况下跳到尽可能高的位置？如果找到了这个位置，它的父亲就是LCA了
        if(fa[x][k]!=fa[y][k]) //如果发现x,y节点还没有上升到最近公共祖先节点，公共祖先节点以上的节点都是相同的
        {
            x=fa[x][k];//当然要跳跃
            y=fa[y][k];//当然要跳跃
        }
    return fa[x][0];//必须返回x的父亲节点,也就是Lca(x,y)
}

```

## Tarjan 算法