---
layout: post
title: Unique Binary Search Trees
description: "Unique Binary Search Trees"
modified: 2014-05-29
tags: [Leetcode, Algorithm]
categories: [Leetcode, Algorithm]
---

## 题目描述
1.Given n, how many structurally unique BST's (binary search trees) that store values 1...n?
For example, Given n = 3, there are a total of 5 unique BST's.

{% highlight C++ %}
   1         3     3      2      1
    \       /     /      / \      \
     3     2     1      1   3      2
    /     /       \                 \
   2     1         2                 3
{% endhighlight %}

2.Given n, generate all structurally unique BST's (binary search trees) that store values 1...n?

一般认清问题比解决问题更重要，`问题的解=事实+规则`，事实就是那些静态知识，规则就是知识间的关系。这道题第一问是给定序列[1, n]，求解出其所有BST的个数，第二问则生成这些BST。这边的序列就可以看成是事实，BST则是规则，对问题的解进行了约束。
具体点来分析规则，BST是二叉查找树，也就意味着`左子树根节点的值（或为空） < 根节点的值 < 右子树根节点的值（或为空）`，第一问要求出所有可能的BST，很明显，序列[start, end]中每一个节点都可以作为根节点，若假设节点i为根节点，则其左子树由[start, i-1]组成，其右子树由[i+1, end]构成，如此就把问题分解成相同的两个子问题，得出如下公式：

> 以i为根节点的BST数量=左子树BST数量*右子树BST数量

但是这两个子问题的求解是有重复的。可以看到一个区间的BST的数量只和区间长度有关，若两个子问题的区间长度一样，那么其BST的数量也是一样的。重复子问题就很自然的想到动态规划，而动态规划关键就是要记录之前已经计算过的子问题的解：记f(i)为区间长度为i的BST数量，则动态规划方程为

$$ f(i)=\sum_{j=1}^{i} f(j-1)*f(i-j) $$

至此，问题全部分析完，剩下就是编码的事了。

## 代码实现
{% highlight C++ %}
class Solution {
public:
    int numTrees(int n) {
        vector<int> dp(n+1, 0);
        dp[0] = 1;
        dp[1] = 1;
        for (int i = 2; i < n+1; ++i){
            for (int j = 0; j < i; ++j){
                dp[i] += dp[j-0] * dp[i-j-1];
            }
        }
        return dp[n];
    }
};
{% endhighlight %}

就短短10多行代码！那么再考虑第二个问题，要生成这些BST，思想过程其实和上面一样，注意到BST是具有递归结构的，考虑采用递归实现，递归函数的输入即为一个区间[start, end]，输出为此区间长度的BSTs（用vector作为容器），递归出口为start>end（此时表现为左子树为空或者右子树为空），代码如下：

{% highlight C++ %}
/**
 * Definition for binary tree
 * struct TreeNode {
 *     int val;
 *     TreeNode *left;
 *     TreeNode *right;
 *     TreeNode(int x) : val(x), left(NULL), right(NULL) {}
 * };
 */
class Solution {
public:
    vector<TreeNode *> generateTrees(int n) {
        if (n == 0) _generateTrees_aux(1,0);
        return _generateTrees_aux(1, n);
    }
    
    vector<TreeNode *> _generateTrees_aux(int start, int end){
        vector<TreeNode *> result;
        if (start > end){
            result.push_back(nullptr);
            return result;
        }
        for (int i  =start; i <= end; ++i){
            vector<TreeNode *> leftsubtree = _generateTrees_aux(start, i-1);
            vector<TreeNode *> rightsubtree = _generateTrees_aux(i+1, end);
            for (auto left_root:leftsubtree){
                for (auto right_root:rightsubtree){
                    TreeNode *root = new TreeNode(i);
                    root->left = left_root;
                    root->right = right_root;
                    result.push_back(root);
                }
            }
        }
        return result;
    }
};
{% endhighlight %}