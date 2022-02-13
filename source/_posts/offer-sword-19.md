---
layout:     post
title:      "「剑指 Offer」面试题 19：二叉树的镜像"
date:       2019-01-10 03:00:00
author:     "Ink Bai"
header-img: "/img/post/offer-sword-19.jpg"
catalog:    true
tags:
    剑指 Offer
---

> 请完成一个函数，输入一个二叉树，该函数输出它的镜像。

一颗树的镜像的过程是：前序遍历这棵树的每个结点，如果遍历到的结点有子结点，就交换这两个结点，然后再递归得出这两个子结点的镜像，递归的终止条件是已经到达了叶子结点，就得到了整个树的镜像，代码如下：

```java
/**
public class TreeNode {
    int val = 0;
    TreeNode left = null;
    TreeNode right = null;

    public TreeNode(int val) {
        this.val = val;

    }

}
*/
public class Solution {
    public void Mirror(TreeNode root) {
        if (root == null) return;
        if (root.left == null && root.right == null) return;

        TreeNode temp = root.left;
        root.left = root.right;
        root.right = temp;

        if (root.left != null) {
            Mirror(root.left);
        }
        if (root.right != null) {
            Mirror(root.right);
        }
    }
}
```
