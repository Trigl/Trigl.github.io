---
layout:     post
title:      "「剑指 Offer」面试题 25：二叉树中和为某一值的路径"
date:       2019-03-11 02:00:00
author:     "Ink Bai"
header-img: "/img/post/offer-sword-25.jpg"
tags:
    - 面试
---

> 输入一颗二叉树和一个整数，打印出二叉树中结点值的和为输入整数的所有路径。从树的根结点开始往下一直到叶结点所经过的结点形成一条路径。

用递归来做，处理某个结点时，首先将这个结点放入到代表路径的 list 里面，如果这个结点的值等于 target 值并且还是叶子结点，那么这个路径就是要找的路径，否则继续递归遍历其左右结点，当遍历完左右结点以后，从代表路径的 list 中移除这个结点，也就是代表回到树的上一层换一条路遍历。

```java
import java.util.ArrayList;
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
    ArrayList<ArrayList<Integer>> result = new ArrayList<ArrayList<Integer>>();
    ArrayList<Integer> path = new ArrayList<Integer>();

    public ArrayList<ArrayList<Integer>> FindPath(TreeNode root, int target) {
        if (root != null) {
            // 首先将要处理的这个结点加入 path 中
            path.add(root.val);
            target -= root.val;

            if (target == 0 && root.left == null && root.right == null) {
                // 这里不要用 result.add(path)，因为最终所有 path 都会指向同一个引用，最终结果为空
                result.add(new ArrayList<Integer>(path));
            }

            FindPath(root.left, target);
            FindPath(root.right, target);

            // 处理结束后把这个结点移除
            path.remove(path.size() - 1);
        }

        return result;
    }
}
```
