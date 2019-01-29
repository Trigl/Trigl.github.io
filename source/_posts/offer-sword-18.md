---
layout:     post
title:      "「剑指 Offer」面试题 18：树的子结构"
date:       2019-01-10 02:00:00
author:     "Ink Bai"
header-img: "/img/post/offer-sword-18.jpg"
tags:
    - 面试
---

> 输入两颗二叉树 A 和 B，判断 B 是不是 A 的字结构。（ps：我们约定空树不是任意一个树的子结构）

要查找树 A 中是否存在和树 B 结构一样的子树，我们可以分成两部：第一步在树 A 中找到和 B 的根结点的值一样的结点 R，第二步再判断树 A 中以 R 为根结点的子树是不是含有和树 B 一样的结构。

第一步在树 A 中查找与根结点的值一样的结点，这实际上就是树的遍历，可以用递归的方法来实现。

第二步是判断树 A 中以 R 为根结点的子树是不是和树 B 具有相同的结构。同样，我们也可以用递归的思路来考虑：如果结点 R 的值和树 B 的根结点不相同，则以 R 为根结点的子树和树 B 肯定不具有相同的结点；如果它们的值相同，则递归地判断它们各自的左右结点的值是不是相同。递归的终止条件是我们到达了树 A 或者树 B 的叶结点。

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
        boolean result = false;
        if (root1 != null && root2 != null) {
            if (root1.val == root2.val) {
                result = DoesTree1HasTree2(root1, root2);
            }
            if (!result) {
                result = HasSubtree(root1.left, root2) || HasSubtree(root1.right, root2);
            }
        }
        return result;
    }

    private boolean DoesTree1HasTree2(TreeNode tree1, TreeNode tree2) {
        if (tree2 == null) {
            return true;
        }
        if (tree1 == null) {
            return false;
        }
        if (tree1.val != tree2.val) {
            return false;
        }
        return DoesTree1HasTree2(tree1.left, tree2.left) && DoesTree1HasTree2(tree1.right, tree2.right);
    }
}
```
