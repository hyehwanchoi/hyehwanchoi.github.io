---
layout: single
title:  "Binary Tree Inorder Traversal"
date:   2019-11-10 21:00:00
author: HyeHwan Choi
categories: algorithm
tags:   algorithm
---

# 이진트리 중위순회

Difficulty: Medium  

Question)  
Given a binary tree, return the inorder traversal of its nodes' values.  
이진트리가 주어지면, 노드값의 중위순위를 돌려줍니다.

**Example :**
```
Input: [1,null,2,3]
1 level   1 (root node)
   |       \ (edge)
2 level     2 (internal node)
   |       /
3 level   3 (leaf node)
Output: [1,3,2]
```
      

문제를 풀기 위해 트리와 이진트리(binary tree)에 대해서 알아보겠습니다.    

최상단의 노드를 root라고 합니다.  
여기서 노드란 위에서 보듯이 1,2,3 등 edge와 edge 사이를 얘기합니다.  
루트 노드(root node)는 부모가 없는 최상단의 노드이며, 잎새 노드(leaf node)는 
자식 노드가 없는 노드입니다.  
그 외에 노드들을 internal node라고 합니다.    

이진트리(binary tree) 란 자식노드가 최대 두 개인 노드로 구성된 트리입니다.  
이진트리의 종류에는 정이진트리(full binary tree), 완전이진트리(complete binary tree), 
균형이진트리(balanced binary tree) 등이 있습니다.    

그리고 중위순회(inorder traversal)란 탐색순회 방법으로 왼쪽 서브트리 ->  
노드 -> 오른쪽 서브트리 순으로 순회하는 방식입니다.

**Solution :**
```
/**
 * Definition for a binary tree node.
 * public class TreeNode {
 *     int val;
 *     TreeNode left;
 *     TreeNode right;
 *     TreeNode(int x) { val = x; }
 * }
 */
class Solution {
    List<Integer> returnList = new ArrayList<Integer>();
    public List<Integer> inorderTraversal(TreeNode root) {
        if(root != null){
            inorderTraversal(root.left); // 루트 노드의 왼쪽으로 이동
            returnList.add(root.val);
            inorderTraversal(root.right); // 루트 노드의 오른쪽으로 이동
        }
        
        return returnList;
    }
}
```