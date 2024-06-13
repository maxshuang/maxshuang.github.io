```
#include <queue>
#include <vector>
#include <functional>
#include <map>
#include <string>
#define CATCH_CONFIG_MAIN
#include "catch.hpp"

// HuffmanTree is for compression codec.
// The abstracted problem is "Weighted Path Length of Tree，WPL"

class HuffmanTree {
private:
    struct TreeNode {
        int weight;
        std::string str;
        TreeNode* left;
        TreeNode* right;

        TreeNode(int w=0, const std::string& s="", TreeNode* l=nullptr, TreeNode* r=nullptr): weight(w), str(s), left(l), right(r){}

        bool operator()(const TreeNode& rh) const {
            return this->weight < rh.weight;
        }
    };

    struct NodeGreater{
        bool operator()(const TreeNode* lf, const TreeNode* rh) const {
            return !(lf->operator()(*rh));
        }
    };

    // std::function<[](const TreeNode* lf, const TreeNode* rh){return std::greater<TreeNode>(*lf, *rh)();}>

    typedef std::priority_queue<TreeNode*, std::vector<TreeNode*>, NodeGreater> MINPQ;    
    
    TreeNode* tree=nullptr;
    MINPQ pq;
    int minWPL=0;
    std::map<std::string, std::string> codecTable;

private:
    TreeNode* buildTree(const std::map<std::string, int>& table) {
        for (const auto& pair : table) {
            TreeNode* node = new TreeNode(pair.second, pair.first);
            pq.push(node);
        }

        int cn=table.size();
        while(--cn) {
            TreeNode* lf=pq.top();
            pq.pop();
            TreeNode* rh=pq.top();
            pq.pop();
            pq.push(mergeTree(lf, rh));
        }

        return pq.top();
    }

    TreeNode* mergeTree(TreeNode* lf, TreeNode* rh) {
        TreeNode* node = new TreeNode(lf->weight+rh->weight, "", lf, rh);
        return node;
    }

    // dfs: O(N)
    void getWPL(const TreeNode* root) {
        if(!root) return;

        // dfs
        minWPL=dfsWPL(root, 0);
    }

    int dfsWPL(const TreeNode* root, int depth) {
        if(!root) return 0;
        // leave node
        if(!root->left && !root->right) return root->weight*depth;

        return dfsWPL(root->left, depth+1)+dfsWPL(root->right, depth+1);
    }

    // dfs: O(N)
    void genCodecTable(const TreeNode* root) {
        std::string codec;
        dfsCodec(root, codec);
    }

    void dfsCodec(const TreeNode* root, std::string& codec) {
        if(!root) return;
        // leave node
        if(!root->left && !root->right) {
            codecTable[root->str]=codec;
            codec.pop_back();
            return;
        }

        codec.push_back('0');
        dfsCodec(root->left, codec);
        codec.push_back('1');
        dfsCodec(root->right, codec);

        // revert back
        codec.pop_back();
    }

public:
    // O(N)
    HuffmanTree(const std::map<std::string, int>& table) {
        tree=buildTree(table);
        getWPL(tree);
        genCodecTable(tree);
    }

    ~HuffmanTree() {
        // [TODO] delete tree
    }

    int getMinWPL() const {
        return minWPL;
    }

    std::map<std::string, std::string> getCodecTable() const {
        return codecTable;
    }
};

// Quite impressive !!!
// the minWPL is the sum of all non-leave nodes, you can imagine that the $depth*leave node's weight=$ splits to every non-leave nodes
int getMinWPL(int arr[], int n) {
  std::priority_queue<int, std::vector<int>, std::greater<int>> huffman;
  for (int i = 0; i < n; i++) huffman.push(arr[i]);

  int res = 0;
  for (int i = 0; i < n - 1; i++) {
    int x = huffman.top();
    huffman.pop();
    int y = huffman.top();
    huffman.pop();
    int temp = x + y;
    res += temp;
    huffman.push(temp);
  }
  return res;
}

TEST_CASE("Test Huffman Tree", "") {
    std::map<std::string, int> table={{"A", 35}, {"B", 25}, {"C", 15}, {"D", 13}, {"E", 10}};
    HuffmanTree hfTree(table);

    REQUIRE(hfTree.getMinWPL()==219);
    std::map<std::string, std::string> tmp={{"A", "11"}, {"B", "10"}, {"C", "00"}, {"D", "011"}, {"E", "010"}};
    REQUIRE(hfTree.getCodecTable()==tmp);
}
```