---
title: "leetcode 337 House Robber III"
date: 2021-05-31T00:36:26+08:00 
toc: true 
tags:
- go
- leetcode

---
### 介绍

![image](https://image.syst.top/image/leetcode/337.jpg)

这是 House Robber III。

这道题和之前两个版本最大不同在于引入了二叉树。
树的每个节点(具体的房子)都有对应的值(金额)。每个节点有两种状态，偷或者不偷。
规则是不能同时偷父子节点，
![image](https://image.syst.top/image/leetcode/337-2.png)

比如上图，如果选择偷 `A` 节点，那么 `B` 和 `C` 就不能偷，既然 `B` 和 `C` 不能偷，那么 `DEFG` 就可以偷的。
因为 `DEFG` 和 `A` 是子孙关系，而不是父子关系。

这样的背景下，求小偷能偷的最大金额?

### 暴力递归

我们还是用上图的那个例子。我们可以直接求出最大金额,伪代码如下

```go

/**
 * Definition for a binary tree node.
 * type TreeNode struct {
 *     Val int
 *     Left *TreeNode
 *     Right *TreeNode
 * }
 */

// 求当前A节点能偷的最大金额res
// 伪代码 
item := node.Val + node.Left.Left.Val + node.Left.Right.Val +
		node.Right.Left.Val + node.Right.Right.Val
res := max(item, node.Left.Val+node.Right.val)
```

实际中，`DE` 是 `B` 的子节点，`DE` 也会变成别人的父节点，甚至是爷爷节点。

对于任意一个节点，求对应能偷的最大值本质就是上述的代码，我们需要改成递推公式。

```go
// 递推公式伪代码
func rob(node *TreeNode) int {
	item1 := node.Val + rob(node.Left.Left) + rob(node.Left.Right) +
		rob(node.Right.Left) + rob(node.Right.Right)

	res := max(item1, rob(node.Left)+rob(node.Right))
}
```

然后我们就可以翻译上面的代码,

```go
/**
 * Definition for a binary tree node.
 * type TreeNode struct {
 *     Val int
 *     Left *TreeNode
 *     Right *TreeNode
 * }
 */

func rob(root *TreeNode) int {
	if root == nil {
		return 0
	}
	
	money := root.Val
	
	if root.Left != nil {
		money += (rob(root.Left.Left) + rob(root.Left.Right))
	}

	if root.Right != nil {
		money += (rob(root.Right.Left) + rob(root.Right.Right))
	}

	res := max(money, rob(root.Left)+rob(root.Right))
	return res
}

func max(item1, item2 int) int {
	if item1 >= item2 {
		return item1
	}
	return item2
}
```
要是这样直接运行，Leetcode 一般会报以下错误。
![image](https://image.syst.top/image/leetcode/337-3.png)

存在大量重复计算节点。比如在计算 `A` 节点的值时，会计算`BC`，同时也会计算 `DEFG` 的值。
这种情况下，当计算 `BC` 的时候，又会重新计算一遍 `DEFG`......

没什么是加缓存解决不了的。

### 记忆化递归

它也有个高大上的名字，叫记忆化递归。

```go
/**
 * Definition for a binary tree node.
 * type TreeNode struct {
 *     Val int
 *     Left *TreeNode
 *     Right *TreeNode
 * }
 */

var cache map[*TreeNode]int 

func init() {
	cache = make(map[*TreeNode]int)
}

func rob(root *TreeNode) int {
	if root == nil {
		return 0
	}

	if item, ok := cache[root]; ok {
		return item
	}

	money := root.Val

	if root.Left != nil {
		money += (rob(root.Left.Left) + rob(root.Left.Right))
	}

	if root.Right != nil {
		money += (rob(root.Right.Left) + rob(root.Right.Right))
	}

	res := max(money, rob(root.Left)+rob(root.Right))
	cache[root] = res
	return res
}

func max(item1, item2 int) int {
	if item1 >= item2 {
		return item1
	}
	return item2
}
```

那么，如何用动态规划的思想解这道题？

动态规划最重要的两步:
- 定义状态
- 状态转移方程

首先是状态。
针对每一个节点，都只有两种状态，偷或者不偷。
在这个基础上，我们可以进一步的保存每一个状态偷或者不偷情况下当前最大金额值。

假设当前是q节点。
我们用 q[0] 表示选择偷 `q` 节点时当前最大金额值。用 `q[1]` 表示不选择 `q` 节点的情况下当前最大金额值。即:
```go
// 伪代码
q[0]=xxx
q[1]=xxx
```
那么最终我们只需要:
```go
// 伪代码
max(q[0],q[1])
```

接着就是状态转移方程。

假设 `l` 和 `r` 分别代表 `q` 的左右子节点。 那么，就会有以下公式:

 当偷 `q` 时，`q` 的左右子节点不能被偷，那么偷 `q` 情况下最大总金额= `q` 自身的值 +不能被偷的 `l` 和 `r` 位置最大金额值的和。即，
```go
// 伪代码
q[0]=q.val+l[1]+r[1]
```

当不偷 `q` 时，`q` 的左右子节点都能偷。但是不一定要去偷。这取决于要偷和不偷情况下哪个金额更多。即:
```go
// 伪代码
q[1]=max(l[0],l[1])+max(r[0],r[1])
```

这样的话就可以翻译代码了。
```go
/**
 * Definition for a binary tree node.
 * type TreeNode struct {
 *     Val int
 *     Left *TreeNode
 *     Right *TreeNode
 * }
 */
func rob(root *TreeNode) int {
	val := df(root)
	return max(val[0], val[1])
}

func df(root *TreeNode) [2]int {
	var res [2]int
	// 递归出口
	if root == nil {
		return [2]int{0, 0}
	}

	l, r := df(root.Left), df(root.Right)
	// 选中当前节点情况下最大值
	res[0] = root.Val + l[1] + r[1]
	// 不选择当前节点情况下最大值
	res[1] = max(l[0], l[1]) + max(r[0], r[1])
	return res
}

func max(item1, item2 int) int {
	if item1 >= item2 {
		return item1
	}
	return item2
}

```













