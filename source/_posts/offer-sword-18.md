---
layout:     post
title:      "「剑指 Offer」面试题 18：树的子结构"
date:       2019-01-10 02:00:00
author:     "Ink Bai"
header-img: "/img/post/offer-sword-18.jpg"
tags:
    剑指 Offer
---

> 输入两颗二叉树 A 和 B，判断 B 是不是 A 的字结构。（ps：我们约定空树不是任意一个树的子结构）

要查找树 A 中是否存在和树 B 结构一样的子树，这就涉及到一个树的遍历问题，每遍历到一个结点我们就需要判断 B 是否是这个结点所在的子树，如果是的话直接返回，如果不是的话就要继续遍历这个结点的左右子结点。

所以我们需要两个函数，第一个函数控制总的遍历逻辑，第二个函数要判断一个子树是否是另一个子树的一部分，但是与第一个函数不同的是第二个子树不需要遍历，直接从根结点就开始比较，根结点相同的话继续比较这两个树的左右子结点，所以也可以用递归来实现，递归结束的条件就是某个树到达了其跟结点。

综上，代码如下：

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
    public boolean HasSubtree(TreeNode root1,TreeNode root2) {
        if (root1 == null || root2 == null) {
            return false;
        }

        boolean result = isSubTree(root1, root2);

        if (!result) {
            result = HasSubtree(root1.left, root2) || HasSubtree(root1.right, root2);
        }

        return result;
    }

    private boolean isSubTree(TreeNode node1, TreeNode node2) {
        if (node2 == nulls) {
            return true;
        }

        if (node1 == null) {
            return false;
        }

        if (node1.val != node2.val) {
            return false;
        }

        return isSubTree(node1.left, node2.left) & isSubTree(node1.right, node2.right);
    }
}
```
