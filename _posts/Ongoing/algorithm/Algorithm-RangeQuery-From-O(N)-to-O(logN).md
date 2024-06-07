

## Traversal 
O(N)



## Block Decomposition
![alt text](image.png)

对于这种区间求和问题，最直观且最正常的想法就是遍历。将操作独立开来看:  
1. 对于区间变更问题，使用差分数组的形式可以将 $O(N)$ 问题转换成 $O(1)$ 问题
2. 对于区间求和问题，可以使用前缀和数组的形式将 $O(N)$ 问题转换成 $O(1)$ 问题

但是对于交替操作，就需要存储更多额外的信息以便复用历史结果，降低时间复杂度。

解决这个问题的 intuition 的 2 个：
1. 区间加值或者区间减值如果要应用在区间求和上要转换成一个 lazy tag，延迟操作。 
2. 为了实现快速的区间求和，可以使用一些已经保存过的区间和，然后再上面应用 lazy tag, 就可以复用之前的计算结果。

将这些 intuition 转变成 Block Decomposition 算法，就是预先对数组进行分块，然后维护分块的区间求和。当求解任意区间求和时，它可以转变成多个分块区间和的加法操作。

![alt text](image-1.png)

由于实际执行的区间加值操作可能只覆盖部分分块区间，对于这种情况，需要转换成复杂度为 $O(s)$ 的遍历操作。对于全区间覆盖，操作复杂度为 $O(1)$，可能覆盖的全区间个数为 $O(n/s)$。所以总的操作时间复杂度为 $O(n/s+s)$，当 $s=\sqrt{n}$ 时可以取到最小值。

```
#include <vector>

// two operations:
// 1. add, l, r, x
// 2. sum, l, r

enum Type {
    ADD,
    SUM,
};

struct Op {
    Type tp;
    int l;
    int r;
    int x;
};

int blockSize=0;
int blockCount=0;
std::vector<int> blockSum;  // for range sum
std::vector<int> lazyTag;   // for range add/sub

void add(std::vector<int>& array, const Op& op);
int sum(std::vector<int>& array, const Op& op);

std::vector<int> Calculate(std::vector<int>& array, const std::vector<Op>& ops) {
    // block decomposite the array
    int arrSize=array.size();
    blockSize=std::sqrt(arrSize);
    blockCount=(arrSize+blockSize-1)/blockSize;
    blockSum.resize(blockCount, 0);
    lazyTag.resize(blockCount, 0);
    for(int i=0; i<blockCount; ++i) {
        int start=blockSize*i;
        int end=std::min(blockSize, arrSize-start);
        for(int j=0; j<end; ++j) {
            blockSum[i]+=array[start+j];
        }
    }

    // do the operations
    std::vector<int> res;
    for(size_t i=0; i<ops.size(); ++i) {
        switch(ops[i].tp) {
            case ADD:
                add(array, ops[i]);
                break;
            case SUM:
                res.push_back(sum(array, ops[i]));
                break;
        }
    }

    return res;
}

// for Add:
// 1. cover the whole block, add the op.x*blockSize to the sum
// 2. cover only a part of block, do the range add
void add(std::vector<int>& array, const Op& op) {
    int lb=op.l/blockSize, rb=op.r/blockSize;

    // inside a single block
    if(lb==rb) {
        for(int i=op.l; i<=op.r; ++i) {
            array[i]+=op.x;
        }
        blockSum[lb]+= (op.r-op.l+1)*op.x;
        return;
    }

    // [l, (lb+1)*blockSize) 
    for(int i=op.l; i<(lb+1)*blockSize; ++i) {
        array[i]+=op.x;
    }
    blockSum[lb]+=((lb+1)*blockSize-op.l)*op.x;
    
    // [(lb+1)*blockSize, rb*blockSize)
    for(int i=lb+1; i<rb; ++i) {
        blockSum[i]+=blockSize*op.x;
        lazyTag[i]+=op.x;
    }

    // [rb*blockSize, op.r]
    for(int i=rb*blockSize; i<=op.r; ++i) {
        array[i]+=op.x;
    }
    blockSum[rb]+=(op.r-rb*blockSize+1)*op.x;
}

// for Sum:
// 1. cover the whole block, add block sum to total
// 2. cover a part of block, do the range sum
int sum(std::vector<int>& array, const Op& op) {
    // NOTE: can use static table to accelerate the query
    int lb=op.l/blockSize, rb=op.r/blockSize;
    int total=0;

    // inside a single block
    if(lb==rb) {    
        for(int i=op.l; i<=op.r; ++i) {
            total+=array[i];
        }
        total += lazyTag[lb]*(op.r-op.l+1);
        return total;
    }

    // [l, (lb+1)*blockSize) 
    for(int i=op.l; i<(lb+1)*blockSize; ++i) {
        total+=array[i];
    }
    total += lazyTag[lb]*((lb+1)*blockSize-op.l);
    
    // [(lb+1)*blockSize, rb*blockSize)
    for(int i=lb+1; i<rb; ++i) {
        total+=blockSum[i];
    }

    // [rb*blockSize, op.r]
    for(int i=rb*blockSize; i<=op.r; ++i) {
       total+=array[i];
    }
    total += lazyTag[rb]*(op.r-rb*blockSize+1);

    return total;
}
```

如果是查询多过修改，还可以通过维护分块内和分块间前缀和的方式，将查询复杂度降到 $O(1)$，但是修改的时间复杂度会有常数系数的增加。

除了区间和，分块算法使用于几乎所有的区间操作，比如查找区间中大于某个值的个数，查找区间中满足某个条件的值个数等等。

## Segment Tree 线段树
Segment tree 在设计上的 intuition 非常直观，它本质上就是*分块* + *信息复用*。

和我们最开始分块思想非常类似，但是为了避免 $O(\sqrt{N})$ 级别的分块更新，它需要使用更多的内存信息做跨块之间的覆盖，这样就可以更快的定位到变更区间。

所以你可以看到 segment tree 在分块和信息复用上做的非常彻底，将所有的区间不断二分直到只剩一个元素(叶子节点)，所有的非叶子节点都是叶子节点的分层信息复用。

线段树的形态的例子：
![alt text](image-3.png)

线段树通过二分的方式不断划分节点，形成一颗 complete tree，它的树高为 $O(\lceil logN \rceil)$，满节点的情况下节点数量为$O(2^{\lceil logN \rceil+1}-1)\approx4n-5$。所以可以认为它的空间复杂度在 $O(4N)$ 这个级别。

对于时间复杂度，我们可以看到：
1. 对于所有的区间更新，都可以通过一个 lazy tag 在上层的覆盖区间上操作，复杂度为 $O(logN)$。
2. 对于所有的区间查询，可以从 root 节点到叶子节点的路径上找到覆盖区间，由于树高为 $O(\lceil logN \rceil)$，所以查询复杂度也是 $O(logN)$。

### 树的创建
如果已经知道原始数组的大小，我们可以直接定义一个 4n 大小的树节点数组或者使用动态 new 的方式构建 segment tree。

树节点数组方式，内存更加紧凑，locality 特性更好。

```
struct SegmentTree {
    const static int MAXNODE=100;
    // complete tree of MAXNODE leaf nodes has 4*MAXNODE-5 nodes
    int nodes[4*MAXNODE];
    int size;

    SegmentTree(const std::vector<int>& arr) {
        size=arr.size();
        // we use node 1 as the root node, so that its left child is 2*p, right child is 2*p+1
        build(arr, 0, arr.size()-1, 1); 
    }

    void build(const std::vector<int>& arr, int s, int t, int p) {
        // we meet the leaf node here
        if (s == t) {
            nodes[p] = arr[s];
            return;
        }

        // divide from the medium
        int m = s + ((t - s) >> 1);
        build(arr, s, m, p * 2);
        build(arr, m + 1, t, p * 2 + 1);

        // nodes[i] is the sum for range here
        nodes[p] = nodes[p * 2] + nodes[(p * 2) + 1];
    }

    //...
};
```

动态 new 的方式：
```
#include <vector>

struct Range {
    int l;
    int r;

    Range(int l, int r): l(l), r(r){}
};

struct TreeNode {
    TreeNode* left;
    TreeNode* right;
    int sum;
    Range range;

    TreeNode(TreeNode* left, TreeNode* right, Range r): left(left), 
        right(right), sum(0), range(r){}
};

struct SegmentTree {
    TreeNode root;

    SegmentTree(const std::vector<int>& arr): root(nullptr, nullptr, Range{0, arr.size()-1}) {
        build_dfs(&root, arr, 0, arr.size()-1);
    }

    void build_dfs(TreeNode* root, const std::vector<int>& arr, int l, int r) {
        if(l==r) {
            root->sum=arr[l];
            return;
        }

        int m = l + ((r - l) >> 1);
        root->left=new TreeNode(nullptr, nullptr, Range{l, m});
        root->right=new TreeNode(nullptr, nullptr, Range{m+1, r});
        build_dfs(root->left, arr, l, m);
        build_dfs(root->right, arr, m+1, r);

        root->sum=root->left->sum + root->right->sum;
    } 
};
```

### 区间查询
线段树的区间查询也好理解，从 root 节点出发，如果查询区间不能覆盖节点区间，说明查询区间只是节点区间的一部分，我们需要去更小的左右子节点区间上查找，直到我们找到了查询区间可以完全覆盖节点区间的节点。

```
int getsum(int l, int r, int s, int t, int p) {
  if (l <= s && t <= r)
    return d[p];
  int m = s + ((t - s) >> 1), sum = 0;
  if (l <= m) sum += getsum(l, r, s, m, p * 2);
  if (r > m) sum += getsum(l, r, m + 1, t, p * 2 + 1);
  return sum;
}
```

上面是没有考虑区间修改的情况，因为区间修改会涉及到 lazy tag，lazy tag 会影响到某个节点所有的子节点，比如叶子节点的查询结果。

### 区间修改
区间修改和区间查询类似，不同点只是如果找到查询区间可以覆盖节点区间的节点，直接将区间修改保存到 lazy tag 中即可，不用再继续传播，通过这种方式实现 $O(logN)$ 的修改时间复杂度。

实现过程中有个优化细节，就是每次在做区间修改的时候，如果在检测到当前节点已经存在 lazy tag，可以下推给子节点，这样可以通过均摊的方式减少查询过程中需要多次叠加中间节点的 lazy tag。

```
void updateRange(int l, int r, int v) {
    return updateRangeRecur(l, r, v, 0, size-1, 1);
}

// [l, r] is the update range, [s, t] is the current range
void updateRangeRecur(int l, int r, int v, int s, int t, int p) {
    if(l<=s && t<=r) {
        // cover range
        nodes[p].sum+=(t-s+1)*v;
        nodes[p].lazyTag+=v;
        return;
    }

    // interval range
    int m=s+(t-s)/2;
    // push down lazy tag to its children nodes
    // safeguard the leaf node
    if(nodes[p].lazyTag != 0 && s!=t) {
        nodes[2*p].sum=(m-s+1)*nodes[p].lazyTag, nodes[2*p].lazyTag+=nodes[p].lazyTag;
        nodes[2*p+1].sum=(t-m)*nodes[p].lazyTag, nodes[2*p+1].lazyTag+=nodes[p].lazyTag;
        nodes[p].lazyTag=0;
    }

    // |___________|
    // s           t
    //    |____|
    //    l    r
    if(l<=m) updateRangeRecur(l, r, v, s, m, p);
    if(m<=r) updateRangeRecur(l, r, v, m+1, t, p);
    // we need to update current node according to the aboved partial update 
    d[p] = d[p * 2] + d[p * 2 + 1];
    return;
}
```

### 带 lazy tag 的区间查询
```
int querySum(int l, int r) {
    return querySumRecur(l, r, 0, size-1, 1);
}

// [l, r] is the query range, [s, t] is the current range
int querySumRecur(int l, int r, int s, int t, int p) {
    if(l<=s && t<=r) {
        return nodes[p].sum;
    }

    int m = s + ((t - s) >> 1);
        // push down lazy tag to its children nodes
    if(nodes[p].lazyTag != 0) {
        nodes[2*p].sum=(m-s+1)*nodes[p].lazyTag, nodes[2*p].lazyTag+=nodes[p].lazyTag;
        nodes[2*p+1].sum=(t-m)*nodes[p].lazyTag, nodes[2*p+1].lazyTag+=nodes[p].lazyTag;
        nodes[p].lazyTag=0;
    }

    int sum=0;
    if(l<=m) sum+=querySumRecur(l, r, s, m, 2*p);
    if(m<=r) sum+=querySumRecur(l, r, m+1, t, 2*p+1);
    return sum;
}
```

### 单点更新
和区间更新没有区别，只不过击中的区间是叶子节点。

### 区间更新成某个值
原理和区间更新差不多，只不过保存的 lazy tag 不再是增量更新，而是一个值：
```
void update(int l, int r, int c, int s, int t, int p) {
  if (l <= s && t <= r) {
    d[p] = (t - s + 1) * c, b[p] = c;
    return;
  }
  int m = s + ((t - s) >> 1);
  // 额外数组储存是否修改值
  if (v[p]) {
    d[p * 2] = b[p] * (m - s + 1), d[p * 2 + 1] = b[p] * (t - m);
    b[p * 2] = b[p * 2 + 1] = b[p];
    v[p * 2] = v[p * 2 + 1] = 1;
    v[p] = 0;
  }
  if (l <= m) update(l, r, c, s, m, p * 2);
  if (r > m) update(l, r, c, m + 1, t, p * 2 + 1);
  d[p] = d[p * 2] + d[p * 2 + 1];
}

int getsum(int l, int r, int s, int t, int p) {
  if (l <= s && t <= r) return d[p];
  int m = s + ((t - s) >> 1);
  if (v[p]) {
    d[p * 2] = b[p] * (m - s + 1), d[p * 2 + 1] = b[p] * (t - m);
    b[p * 2] = b[p * 2 + 1] = b[p];
    v[p * 2] = v[p * 2 + 1] = 1;
    v[p] = 0;
  }
  int sum = 0;
  if (l <= m) sum = getsum(l, r, s, m, p * 2);
  if (r > m) sum += getsum(l, r, m + 1, t, p * 2 + 1);
  return sum;
}
```

## Fenwick Tree or BIT(Binary Index Tree)

树状数组是另一个通过增加额外的覆盖节点提高算法时间复杂度的数据结构，不同于树状数组，它适用的场景是 **单点修改** 和 **区间查询** 。

我觉得它最大的特点是: 算法的 intuition 很抽象，但是实现超级简单。

到现在为止，我都不理解为什么它会被选择构造成这样子。虽然有一些比较好的[解释](https://cs.stackexchange.com/questions/10538/bit-what-is-the-intuition-behind-a-binary-indexed-tree-and-how-was-it-thought-a)，但我还是很难理解它为什么能够用类似 segment tree 的数据结构构造出 $O(N)$ 空间复杂度且时间复杂度为 $O(logN)$ 的新数据结构。

留做一个 call back 好了，以后有新的想法再回来补充 :)。

我们违反正常的说明顺序，先看下 BIT 的实现。

```
#include <iostream>
#include <vector>
#include "catch.hpp"

using namespace std;

class BIT {
private:
    // start from 1
    vector<int> tree;
    int n;

    int lowBit(int x) {
        return x & -x;
    }

    int getSum(int index) {
        int sum = 0;
        while (index > 0) {
            sum += tree[index];
            // move to its children node
            index -= lowBit(index);
        }
        return sum;
    }

    void updateTree(int index, int val) {
        while (index <= n) {
            tree[index] += val;
            // move to its father node
            index += lowBit(index);
        }
    }

public:
    BIT(int size) : n(size) {
        tree.resize(n + 1, 0);
    }

    void update(int i, int val) {
        updateTree(i + 1, val);
    }

    // for 1 to i
    int query(int i) {
        return getSum(i);
    }

    int queryRange(int l, int r) {
        return getSum(r) - getSum(l - 1);
    }
};
```

我相信你的内心肯定非常疑惑，就这？ 核心代码只有不到 10 行，却可以实现一个 $O(logN)$ 时间复杂度，$O(N)$空间复杂度的 range query。再看看上面的 block decomposition, 空间复杂度为 $O(\sqrt{N})$，但是核心实现和边界处理要远多于 BIT。

BIT 的设计 intuition 我始终觉得非常难以理解，可以看到的一个点是它使用了前缀和来简化区间操作

![alt text](image-2.png)

https://cs.stackexchange.com/questions/10538/bit-what-is-the-intuition-behind-a-binary-indexed-tree-and-how-was-it-thought-a

非常抽象的数据结构，不知道为什么要这样设计？？？

普通树状数组维护的信息及运算要满足 结合律 且 可差分，如加法（和）、乘法（积）、异或等。

结合律：$(x \circ y) \circ z = x \circ (y \circ z)$，其中 $\circ$ 是一个二元运算符。
可差分：具有逆运算的运算，即已知 $x \circ y$ 和 $x$ 可以求出 $y$。
需要注意的是：

模意义下的乘法若要可差分，需保证每个数都存在逆元（模数为质数时一定存在）；
例如 $\gcd$，$\max$ 这些信息不可差分，所以不能用普通树状数组处理。

事实上，树状数组能解决的问题是线段树能解决的问题的子集：树状数组能做的，线段树一定能做；线段树能做的，树状数组不一定可以。然而，树状数组的代码要远比线段树短，时间效率常数也更小，因此仍有学习价值。

有时，在差分数组和辅助数组的帮助下，树状数组还可解决更强的 区间加单点值 和 区间加区间和 问题。