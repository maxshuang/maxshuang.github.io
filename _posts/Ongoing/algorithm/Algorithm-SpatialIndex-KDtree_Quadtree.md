
## QuadTree
![alt text](image-2.png)


## K-Dimension Tree

k-D Tree(KDT , k-Dimension Tree) 是一种可以 高效处理 $k$ 维空间信息的数据结构。

在结点数 $n$ 远大于 $2^{k}$ 时，应用 k-D Tree 的时间效率很好。

Organization of points, lines, planes, … to
support faster processing
• Applications
– Astrophysical simulation – evolution of galaxies
– Graphics – computing object intersections
– Data compression
• Points are representatives of 2x2 blocks in an
image
• Nearest neighbor search

Tree used to store spatial data.
– Nearest neighbor search.
– Range queries.
– Fast look-up
• k-d tree are guaranteed log2 n depth where n
is the number of points in the set.
– Traditionally, k-d trees store points in dimensional space which are equivalent to vectors
in d-dimensional space.

https://courses.cs.washington.edu/courses/cse373/02au/lectures/lecture22l.pdf

![alt text](image.png)
![alt text](image-1.png)

```
#include <vector>
#include <functional>
#include <limits>
#include <queue>
#include <algorithm>
#define CATCH_CONFIG_MAIN
#include "catch.hpp"

// k-d tree is a special kind binary search tree, but with node as hyperplane in each layer

struct Point {
    std::vector<int> coords;
    int id;

    Point(): coords({}), id(-1){}
    Point(const std::vector<int>& c, int id) : coords(c), id(id) {}

    bool operator<(const Point& other) const {
        return coords[0] < other.coords[0];
    }

    bool operator == (const Point& other) const {
        return this->coords==other.coords;
    }
};

double distance(const Point& p1, const Point& p2) {
    double dist=0.0;
    for(size_t i=0; i<p1.coords.size(); ++i) {
        dist+=std::pow(p1.coords[i]-p2.coords[i], 2);
    }

    return std::sqrt(dist);
}

struct KDNode {
    Point p;
    int axis;
    KDNode* left;
    KDNode* right;

    KDNode(Point p, int axis=0, KDNode* left=nullptr, KDNode* right=nullptr): p(p), axis(axis), left(left), right(right){}
};

struct Bounding {
    int left;
    int right;

    Bounding(): left(std::numeric_limits<int>::min()), 
        right(std::numeric_limits<int>::max()){}

    Bounding(int l, int r): left(l), right(r) {}
};

struct BoundingBox {
    std::vector<Bounding> box;

    BoundingBox(int dims): box(dims){}
    BoundingBox(const Point& p): box(p.coords.size()){
        for(size_t i=0; i<p.coords.size(); ++i) {
            box[i].left=p.coords[i];
            box[i].right=p.coords[i];
        }
    }

    void updateBounding(const Point& p) {
        // update all axis
        for(size_t i=0; i<p.coords.size(); ++i) {
            if(box[i].left > p.coords[i]) {
                box[i].left = p.coords[i];
            }
            if(box[i].right < p.coords[i]) {
                box[i].right = p.coords[i];
            }
        }
    }

    double Distance(const Point& p) const{
        double dist=0.0;
        for(size_t i=0; i<box.size(); ++i) {
            if(p.coords[i]<box[i].left) {
                dist+=std::pow(box[i].left-p.coords[i], 2);
            }else if (p.coords[i]>box[i].right) {
                dist+=std::pow(p.coords[i]-box[i].right, 2);
            }
        }

        return std::sqrt(dist);
    }

    BoundingBox TrimLeft(const Point& p, int axis) const{
        BoundingBox tmp = *this;
        tmp.box[axis].right=p.coords[axis];
        return tmp;
    } 

    BoundingBox TrimRight(const Point& p, int axis) const{
        BoundingBox tmp = *this;
        tmp.box[axis].left=p.coords[axis];
        return tmp;
    }
};

class KDTree {
private:
    KDNode* tree;
    int size;
    int curAxis;
    int axisCnt;
    BoundingBox orgBB;

public:
    // [NOTE]: don't input empty set
    KDTree(std::vector<Point>& arr, int dims): tree(nullptr), size(arr.size()), 
        curAxis(0), axisCnt(dims), orgBB(dims) {
        tree = divideRecur(arr, 0, arr.size()-1, curAxis);
    }

    ~KDTree(){
        // TODO delete all nodes
    }

    void Insert(const Point& p) {
        orgBB.updateBounding(p);
        tree = insertRecur(p, tree, tree->axis);
    }

    // quite similar to binary search tree deletion
    // more complicated
    bool Delete(const Point& p) {
        return deleteRecur(p, tree, tree->axis);
    }

    // Time Complexity: O(N)
    // Nearest Neighbor Search
    // This is the prune branch problem
    // https://courses.cs.washington.edu/courses/cse373/02au/lectures/lecture22l.pdf
    Point NNSearch(const Point& p) const {
        Point nearest;
        double bestDist=std::numeric_limits<double>::max();
        NNSearchRecur(tree, p, tree->axis, orgBB, nearest, bestDist);
        return nearest;
    }

    // K-Nearest Neighbor Search
    // Same as NNSearch, the key point is using kth min to prune branches quickly 
    // O(N)
    typedef std::pair<double, KDNode*> DistNodePair;
    struct DistNodeCmp {
        bool operator()(const DistNodePair& lf, const DistNodePair& rt) const{
            return lf.first < rt.first;
        };
    };
    typedef std::priority_queue<DistNodePair, std::vector<DistNodePair>, 
        DistNodeCmp> MAXPQ;

    std::vector<Point> KNNSearch(const Point& p, int k) const {
        // max heap, maintain pq with k size to prune branches
        MAXPQ pq;
        KNNSearchRecur(tree, p, tree->axis, orgBB, pq, k);
        
        std::vector<Point> knn;
        while(!pq.empty()) {
            knn.push_back(pq.top().second->p);
            pq.pop();
        }
        std::reverse(knn.begin(), knn.end());
        return knn;
    }

    std::vector<Point> GetTree() const {
        std::vector<Point> points;
        getTree(tree, points);

        return points;
    }

    int Size() const {
        return size;
    }

private:
    void NNSearchRecur(KDNode* node, const Point& p, int axis, const BoundingBox& bb, 
        Point& nearest, double& bestDist) const{
        // prune branch
        if(!node || bb.Distance(p) > bestDist) return;
    
        double dist=distance(node->p, p);
        if(dist < bestDist) {
            bestDist=dist;
            nearest=node->p;
        }
    
        // visit the most promising side first
        if(p.coords[axis] < node->p.coords[axis]) {
            NNSearchRecur(node->left, p, nextAxis(axis), bb.TrimLeft(node->p, axis), nearest, bestDist);
            NNSearchRecur(node->right, p, nextAxis(axis), bb.TrimRight(node->p, axis), nearest, bestDist);
        }else {
            NNSearchRecur(node->right, p, nextAxis(axis), bb.TrimRight(node->p, axis), nearest, bestDist);
            NNSearchRecur(node->left, p, nextAxis(axis), bb.TrimLeft(node->p, axis), nearest, bestDist);
        }
    }

    void KNNSearchRecur(KDNode* node, const Point& p, int axis, const BoundingBox& bb, 
        MAXPQ& pq, int k) const{
        if(!node) return;
        // prune branch
        if((int)pq.size()>=k && bb.Distance(p)>pq.top().first) return;

        double dist=distance(node->p, p);
        if((int)pq.size()<k) {
            pq.push(DistNodePair{dist, node});
        }else if(dist < pq.top().first){
            pq.pop();
            pq.push(DistNodePair{dist, node});
        }
    
        // visit the most promising side first
        if(p.coords[axis] < node->p.coords[axis]) {
            KNNSearchRecur(node->left, p, nextAxis(axis), bb.TrimLeft(node->p, axis), pq, k);
            KNNSearchRecur(node->right, p, nextAxis(axis), bb.TrimRight(node->p, axis), pq, k);
        }else {
            KNNSearchRecur(node->right, p, nextAxis(axis), bb.TrimRight(node->p, axis), pq, k);
            KNNSearchRecur(node->left, p, nextAxis(axis), bb.TrimLeft(node->p, axis), pq, k);
        }
    }

    bool deleteRecur(const Point& p, KDNode*& root, int axis) {
        if(!root) return false; // miss

        // hit
        if(root->p==p) {
            #if 0
            // The following is wrong because it violates the axis rule of the layer in binary search tree
            else if(!root->left) {
                KDNode* tmp=root;
                root=root->right;
                delete tmp;
            }else if(!root->right) {
                KDNode* tmp=root;
                root=root->left;
                delete tmp;
            }
            #endif
            if (root->right) {
                KDNode* successor = findMin(root->right, axis);
                // copy the data directly
                root->p=successor->p;
                // deal the right tree
                deleteRecur(successor->p, root->right, nextAxis(axis));
            }else if(root->left) {
                // we can't use findMax(left subtree) here, because it can violate the binary search rule
                // so we still find the findMin(left subtree), and then move the whole left subtree to right subtree
                KDNode* min = findMin(root->left, axis);
                root->p=min->p;
                root->right=root->left;
                root->left=nullptr;
                deleteRecur(min->p, root->right, nextAxis(axis));
            }else {
                // leave node
                delete root;
                root=nullptr;
            }

            return true;
        }

        // dfs
        if(p.coords[axis] < root->p.coords[axis]) {
            return deleteRecur(p, root->left, nextAxis(axis));
        }

        return deleteRecur(p, root->right, nextAxis(axis));
    }

    // [NOTE]: we need to find the min node of the axis in subtree node
    KDNode* findMin(KDNode* node, int axis) {
        if(!node) return nullptr;
        
        // the smaller one could be at both children subtrees 
        // min(findMin(left subtree), findMin(right subtree), root)
        if(node->axis != axis) {
            KDNode* lf=findMin(node->left, axis);
            KDNode* rt=findMin(node->right, axis);
            
            KDNode* min=nullptr;
            if(!lf) min=rt;
            else if(!rt) min=lf;
            else if(lf->p.coords[axis] < rt->p.coords[axis]) min=lf;
            else min=rt;

            if(!min) return node;
            return min->p.coords[axis] < node->p.coords[axis]?min:node;
        }

        // root node or in the left subtree
        if(!node->left) return node;
        return findMin(node->left, axis);
    }

    KDNode* insertRecur(const Point& p, KDNode* root, int axis) {
        if(!root) return new KDNode(p, axis);

        if(p==root->p) {
            // do nothing.
            return root;
        }

        // binary tree, maintain the invariant:
        // 1. coords[axis] in left tree < coords[axis] in root
        // 2. coords[axis] in right tree >= coords[axis] in root
        if(p.coords[axis]<root->p.coords[axis]) {
            root->left=insertRecur(p, root->left, nextAxis(axis));
        }else {
            root->right=insertRecur(p, root->right, nextAxis(axis));
        }

        return root;
    }

    void getTree(KDNode* root, std::vector<Point>& points) const {
        if(!root) return;

        points.push_back(root->p);

        getTree(root->left, points);
        getTree(root->right, points);
    }

    KDNode* divideRecur(std::vector<Point>& points, int l, int r, int axis) {
        if(l>r) return nullptr;

        int mid = l+((r-l)>>1);
        // Find the medium and sort the range roughly
        // Time Complexity: O(N)
        // rearrange the vector so that n_element is at this position
        #if 0
        std::nth_element(points.begin()+l, points.begin()+mid, points.begin()+r+1,
                         [&axis](const Point& a, const Point& b) {
                             return a.coords[axis] < b.coords[axis];
                         });
        #endif

        // pre-order
        if(l<r)
            nth_element(points, l, mid, r, axis);
        KDNode* root = new KDNode(points[mid], axis);
        orgBB.updateBounding(points[mid]);

        // dfs: O(N)
        int nAxis=nextAxis(curAxis);
        root->left=divideRecur(points, l, mid-1, nAxis);
        root->right=divideRecur(points, mid+1, r, nAxis);   

        return root;
    }

    int nextAxis(int curAxis) const {
        return (++curAxis)%axisCnt;
    }

    // The following implementation is for std::n_element
    void nth_element(std::vector<Point>& arr, int l, int n, int r, int axis) {
        int pt=-1;
        while(true) {
            pt=partition(arr, l, r, axis);
            if(pt==n) {
                break;
            }else if(pt<n) {
                l=pt+1;
            }else {
                r=pt-1;
            }
        }
    }

    int partition(std::vector<Point>& arr, int l, int r, int axis) {
        if(l>=r) return l;

        int pivot=selectPivot(arr, l, r, axis);
        // invariant: [l, i) <= pivot, [i, j) unknown, [j, r]>=pivot
        int i=l, j=r+1;
        while(true) {
            while(arr[++i].coords[axis]<pivot) if(i>=r) break;
            while(arr[--j].coords[axis]>pivot) if(j<=l) break;

            if(i>=j) break;
            std::swap(arr[i], arr[j]);      
        }

        std::swap(arr[l], arr[j]);

        return j;    // pivot point
    }

    int selectPivot(const std::vector<Point>& arr, int l, int r, int axis) {
        // [TODO]
        return arr[l].coords[axis];
    }
};

TEST_CASE("Test n_element", ""){
    std::vector<Point> points={
        {std::vector<int>{7, 2}, 1},
        {std::vector<int>{9, 6}, 2},
        {std::vector<int>{8, 1}, 3},
        {std::vector<int>{5, 4}, 4},
        {std::vector<int>{4, 7}, 5},
        {std::vector<int>{2, 3}, 6},
    };

    int m=2;
    std::nth_element(points.begin(), points.begin()+m, points.end(), 
                        [](const Point& a, const Point& b) {
                             return a.coords[0] < b.coords[0];
                        });
    std::vector<Point> res = {
        {std::vector<int>{2, 3}, 6},
        {std::vector<int>{4, 7}, 5},
        {std::vector<int>{5, 4}, 4},
        {std::vector<int>{7, 2}, 1},
        {std::vector<int>{8, 1}, 3},
        {std::vector<int>{9, 6}, 2},

    };
    REQUIRE(points==res);
}

TEST_CASE("Test K-Dimensional tree", "create/insert/delete") {
    std::vector<Point> points={
        {std::vector<int>{7, 2}, 1},
        {std::vector<int>{9, 6}, 2},
        {std::vector<int>{8, 1}, 3},
        {std::vector<int>{5, 4}, 4},
        {std::vector<int>{4, 7}, 5},
        {std::vector<int>{2, 3}, 6},
    };

    KDTree kdTree(points, 2);

    /*
    //        <5, 4>
    //        /    \   
    //    <2, 3>   <7, 2>
    //        \    /     \
    //    <4, 7>  <8, 1>  <9, 6>
    */

   SECTION("create") {
        std::vector<Point> res = {
            {std::vector<int>{5, 4}, 4},
            {std::vector<int>{2, 3}, 6},
            {std::vector<int>{4, 7}, 5},
            {std::vector<int>{7, 2}, 1},
            {std::vector<int>{8, 1}, 3},
            {std::vector<int>{9, 6}, 2},
        };
        REQUIRE(kdTree.GetTree()==res);
   }

    SECTION("insert1") {
        kdTree.Insert({std::vector<int>{4, 1}, 7});
        std::vector<Point> res = {
            {std::vector<int>{5, 4}, 4},
            {std::vector<int>{2, 3}, 6},
            {std::vector<int>{4, 1}, 7},
            {std::vector<int>{4, 7}, 5},
            {std::vector<int>{7, 2}, 1},
            {std::vector<int>{8, 1}, 3},
            {std::vector<int>{9, 6}, 2},
        };
        REQUIRE(kdTree.GetTree()==res);
   }

    SECTION("insert2") {
        kdTree.Insert({std::vector<int>{4, 8}, 7});
        std::vector<Point> res = {
            {std::vector<int>{5, 4}, 4},
            {std::vector<int>{2, 3}, 6},
            {std::vector<int>{4, 7}, 5},
            {std::vector<int>{4, 8}, 7},
            {std::vector<int>{7, 2}, 1},
            {std::vector<int>{8, 1}, 3},
            {std::vector<int>{9, 6}, 2},
        };
        REQUIRE(kdTree.GetTree()==res);
   }

    SECTION("insert3") {
        kdTree.Insert({std::vector<int>{10, 7}, 8});
        std::vector<Point> res = {
            {std::vector<int>{5, 4}, 4},
            {std::vector<int>{2, 3}, 6},
            {std::vector<int>{4, 7}, 5},
            {std::vector<int>{7, 2}, 1},
            {std::vector<int>{8, 1}, 3},
            {std::vector<int>{9, 6}, 2},
            {std::vector<int>{10, 7}, 8}
        };
        REQUIRE(kdTree.GetTree()==res);
   }

    SECTION("insert4") {
        kdTree.Insert({std::vector<int>{7, 1}, 8});
        std::vector<Point> res = {
            {std::vector<int>{5, 4}, 4},
            {std::vector<int>{2, 3}, 6},
            {std::vector<int>{4, 7}, 5},
            {std::vector<int>{7, 2}, 1},
            {std::vector<int>{8, 1}, 3},
            {std::vector<int>{7, 1}, 8},
            {std::vector<int>{9, 6}, 2},        
        };
        REQUIRE(kdTree.GetTree()==res);
   }

    SECTION("insert the same node") {
        kdTree.Insert({std::vector<int>{2, 3}, 6});
        std::vector<Point> res = {
            {std::vector<int>{5, 4}, 4},
            {std::vector<int>{2, 3}, 6},
            {std::vector<int>{4, 7}, 5},
            {std::vector<int>{7, 2}, 1},
            {std::vector<int>{8, 1}, 3},
            {std::vector<int>{9, 6}, 2},       
        };
        REQUIRE(kdTree.GetTree()==res);
    }

    SECTION("delete non-existed node") {
        REQUIRE(kdTree.Delete({std::vector<int>{5, 4}, 4})==true);
        REQUIRE(kdTree.Delete({std::vector<int>{7, 2}, 1})==true);
        REQUIRE(kdTree.Delete({std::vector<int>{11, 2}, 1})==false);
        REQUIRE(kdTree.Delete({std::vector<int>{7, 20}, 1})==false);
    }

   SECTION("delete node with both children") {
        REQUIRE(kdTree.Delete( {std::vector<int>{5, 4}, 4})==true);
        std::vector<Point> res = {
            {std::vector<int>{7, 2}, 1},
            {std::vector<int>{2, 3}, 6},
            {std::vector<int>{4, 7}, 5},
            {std::vector<int>{9, 6}, 2},
            {std::vector<int>{8, 1}, 3},
        };
        REQUIRE(kdTree.GetTree()==res);

        REQUIRE(kdTree.Delete( {std::vector<int>{7, 2}, 1})==true);
        res = {
            {std::vector<int>{8, 1}, 3},
            {std::vector<int>{2, 3}, 6},
            {std::vector<int>{4, 7}, 5},
            {std::vector<int>{9, 6}, 2},
        };
        REQUIRE(kdTree.GetTree()==res);
   }

    SECTION("delete node without children") {
        REQUIRE(kdTree.Delete({std::vector<int>{4, 7}, 5})==true);
        std::vector<Point> res = {
            {std::vector<int>{5, 4}, 4},
            {std::vector<int>{2, 3}, 6},
            //{std::vector<int>{4, 7}, 5},
            {std::vector<int>{7, 2}, 1},
            {std::vector<int>{8, 1}, 3},
            {std::vector<int>{9, 6}, 2},
        };
        REQUIRE(kdTree.GetTree()==res);
    }

    SECTION("delete node with one child") {
        REQUIRE(kdTree.Delete({std::vector<int>{2, 3}, 6})==true);
        std::vector<Point> res = {
            {std::vector<int>{5, 4}, 4},
            //{std::vector<int>{2, 3}, 6},
            {std::vector<int>{4, 7}, 5},
            {std::vector<int>{7, 2}, 1},
            {std::vector<int>{8, 1}, 3},
            {std::vector<int>{9, 6}, 2},
        };
        REQUIRE(kdTree.GetTree()==res);
    }

    /*
    //        <5, 4>
    //        /    \   
    //    <2, 3>   <7, 2>
    //        \    /     \
    //    <4, 7>  <8, 1>  <9, 6>
    */
    SECTION("delete node without right child, but with left child") {
        REQUIRE(kdTree.Delete({std::vector<int>{4, 7}, 5})==true);
        kdTree.Insert({std::vector<int>{4, 1}, 11});
        kdTree.Insert({std::vector<int>{4, 2}, 12});
    /*
    // x:     <5, 4>
    //        /     \   
    // y:   <2, 3>  <7, 2>
    //      /       /     \
    // x: <4, 1>   <8, 1>  <9, 6>
    //      \
    // y:   <4, 2>
    */
        REQUIRE(kdTree.Delete({std::vector<int>{2, 3}, 6})==true);
        /*
        // x:     <5, 4>
        //        /    \   
        // y:   <4, 1>   <7, 2>
        //          \    /     \
        // x:    <4, 2> <8, 1>  <9, 6>
        */
        std::vector<Point> res = {
            {std::vector<int>{5, 4}, 4},
            {std::vector<int>{4, 1}, 11},
            {std::vector<int>{4, 2}, 12},
            {std::vector<int>{7, 2}, 1},
            {std::vector<int>{8, 1}, 3},
            {std::vector<int>{9, 6}, 2},
        };
        REQUIRE(kdTree.GetTree()==res);
    }

    /*
    //        <5, 4>
    //        /    \   
    //    <2, 3>   <7, 2>
    //        \    /     \
    //    <4, 7>  <8, 1>  <9, 6>
    */
    SECTION("Nearest Neighbor Search, on the same side") {
        REQUIRE(kdTree.NNSearch(Point{{4, 6}, 21})==Point{{4, 7}, 5});
    }

    SECTION("Nearest Neighbor Search, not on the same side") {
        REQUIRE(kdTree.NNSearch(Point{{10, 3}, 22})==Point{{8, 1}, 3});
    }

    SECTION("Nearest Neighbor Search, exact point") {
        REQUIRE(kdTree.NNSearch(Point{{7, 2}, 1})==Point{{7, 2}, 1});
    }

    SECTION("K Nearest Neighbor Search, all nodes") {
        REQUIRE(kdTree.KNNSearch(Point{{4, 6}, 21}, kdTree.Size())==
            std::vector<Point>{
                {std::vector<int>{4, 7}, 5},
                {std::vector<int>{5, 4}, 4},
                {std::vector<int>{2, 3}, 6},
                {std::vector<int>{9, 6}, 2},
                {std::vector<int>{7, 2}, 1},
                {std::vector<int>{8, 1}, 3},
            });
    }

    SECTION("K Nearest Neighbor Search, k=3") {
        REQUIRE(kdTree.KNNSearch(Point{{10, 3}, 22}, 3)==
            std::vector<Point>{
                {std::vector<int>{8, 1}, 3},
                {std::vector<int>{9, 6}, 2},
                {std::vector<int>{7, 2}, 1},
            });
    }
}
```