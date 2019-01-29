---
layout:     post
title:      "「剑指 Offer」面试题 6：重建二叉树"
date:       2018-11-29 01:00:00
author:     "Ink Bai"
header-img: "/img/post/offer-sword-6.jpg"
tags:
    - 面试
---

> 输入某二叉树的前序遍历和中序遍历的结果，请重建出该二叉树。假设输入的前序遍历和中序遍历的结果中都不含重复的数字。例如输入前序遍历序列 {1, 2, 4, 7, 3, 5, 6, 8} 和中序遍历序列 {4, 7, 2, 1, 5, 3, 8, 6}，则重建出如下图所示的二叉树并返回它的头结点。

![](/img/content/binary-tree-1.png)

首先了解一下二叉树：有一个根结点，根结点下面最多有两个子结点，分别是左右子结点，二叉输的遍历方式一般有三种：

- 前序遍历：先访问根结点，再访问左子结点，最后访问右子结点。
- 中序遍历：先访问左子结点，再访问根结点，最后访问右子结点。
- 后序遍历：先访问左子结点，再访问右子结点，最后访问根结点。

让我们找一下规律，前序遍历先访问根结点，前序遍历的序列是 {1, 2, 4, 7, 3, 5, 6, 8}，所以可以确定，1 就是根结点的值。

![](/img/content/binary-tree-2.png)

中序遍历是先访问左子结点，再访问根结点，上面我们确定了 1 是根结点，那可以说明 1 前面的数字就是左子结点对应的中序遍历序列，1 后面是右子结点的中序遍历序列。

**中序遍历**

![](/img/content/binary-tree-3.png)

由于左右子结点的数目应该是相同的，我们可以根据中序遍历序列确定前序遍历序列的左右子结点如下图。

**前序遍历**

![](/img/content/binary-tree-4.png)

通过上面的步骤我们已经确定了该二叉树的根结点，左子结点的前序遍历和中序遍历，右子结点的前序遍历和中序遍历，自然可以想到用递归的方法去解决，代码如下：

```java
/**
 * Definition for binary tree
 * public class TreeNode {
 *     int val;
 *     TreeNode left;
 *     TreeNode right;
 *     TreeNode(int x) { val = x; }
 * }
 */

import java.util.Arrays;
public class Solution {
    public TreeNode reConstructBinaryTree(int [] pre, int [] in) {
        TreeNode root = reConstructBinaryTree(pre, 0, pre.length - 1, in, 0, in.length - 1);
        return root;
    }

    private TreeNode reConstructBinaryTree(int [] pre, int startPre, int endPre, int [] in, int startIn, int endIn) {
        // 超出界限没有子结点了
        if(startPre > endPre || startIn > endIn) {
            return null;
        }
        TreeNode root = new TreeNode(pre[startPre]);
        for(int i = startIn; i <= endIn; i++) {
            if(in[i] == pre[startPre]) {
                root.left = reConstructBinaryTree(pre, startPre + 1, startPre + i - startIn, in, startIn, i - 1);
                root.right = reConstructBinaryTree(pre, startPre + i -startIn + 1, endPre, in, i + 1, endIn);
            }
        }
        return root;
    }
}
```

下面解释一下代码，首先我们定义一个递归方法，这个方法会输入 pre 和 in，以及它们分别对应的起始下标，第一次调用就是：

```java
TreeNode root = reConstructBinaryTree(pre, 0, pre.length - 1, in, 0, in.length - 1)
```

递归什么时候结束呢，只要到达最后的叶子结点就结束了，判断的条件就是开始下标必须大于结束下标，不满足就返回 `null`，代表已经没有子结点了。

```java
if(startPre > endPre || startIn > endIn)
```

然后对 `in` 即中序遍历的序列循环，因为这样我们可以找出根结点的位置，从而可以区分出左右子结点，当循环到根结点位置时下标位置如下图所示：

**中序遍历**

![](/img/content/binary-tree-6.png)

**前序遍历**

![](/img/content/binary-tree-5.png)

从中序遍历图中可以得到：
左子结点长度 = i - startIn
右子结点长度 = endIn - i

所以对于左子结点来说：
前序遍历起始下标 = (startPre + 1, startPre + i - startIn)
中序遍历起始下标 = (startIn, i - 1)

对于右子结点来说：
前序遍历起始下标 = (startPre + i - startIn + 1, endPre)
中序遍历起始下标 = (i + 1, endIn)

通过上面分析我们可以实现用很简洁的代码完成了整个递归过程，递归的实现很容易想到，但是动手实现并且简洁地实现却是千难万难，这个时候我们就可以结合画图来快速地解决问题。
