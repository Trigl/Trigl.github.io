---
layout:     post
title:      "「剑指 Offer」面试题 27：二叉搜索树与双向链表"
date:       2019-04-06 02:00:00
author:     "Ink Bai"
header-img: "/img/post/offer-sword-27.jpg"
catalog:    true
tags:
    剑指 Offer
---

> 输入一棵二叉搜索树，将该二叉搜索树转换成一个排序的双向链表。要求不能创建任何新的结点，只能调整树中结点指针的指向。

分析一下题目，二叉搜索树，说明根结点的值大于左结点的值，小于右结点的值。然后说要将二叉搜索树转换成一个排序的双向链表，看到排序，又想起了二叉搜索树的结构特点，自然可以想到用左序遍历的方式遍历二叉树就可以得到排序的结果。最后说明不能创建任何新的结点，就是说不能 new 一个对象。

树的题目，并且还需要遍历，很自然想到了使用递归的方法。我们首先递归得到左子树转换后的双向链表，然后将这个链表和根结点连接起来，再递归得到右子树转换后的双向链表，和前面的双向链表连接起来，这样就得到了最终的结果。

使用递归我们还要考虑中止条件是什么，一般树的遍历只要到叶子结点就可以结束遍历了，而本题目到达叶子结点就意味着左右子树都为空。

代码如下：

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
    public TreeNode Convert(TreeNode root) {
        // 如果结点为空或者左右子树都为空时，直接返回该结点
        if (root == null || (root.left == null && root.right == null)) {
            return root;
        }

        TreeNode node = root;
        if (root.left != null) {
            node = Convert(root.left);
            TreeNode current = node;

            // 遍历得到左双向链表的最后一个结点
            while (current.right != null) {
                current = current.right;
            }

            // 将左双向链表和根结点连接起来
            current.right = root;
            root.left = current;
        }

        // 将根结点和右双向链表连接起来
        TreeNode current = Convert(root.right);
        if (current != null) {
            root.right = current;
            current.left = root;
        }

        return node;
    }
}
```
