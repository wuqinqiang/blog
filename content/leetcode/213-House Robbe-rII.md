---
title: "leetcode 213 House Robber II"
date: 2021-05-16T00:17:52+08:00 
toc: true 
tags:
- go
- leetcode

---
### 介绍

![image](https://image.syst.top/image/leetcode/213.png)

这是 House Robber II，也就是 I 的变型版本。II 和 I 的最大区别在于 II 把房子围城一个圈了.
你可以理解成环形链表。这样对小偷打击还是蛮大的，这意味着第一幢房子和最后一幢房子是紧挨着的。不能同时偷。

之前我们提到，相邻两家不能同时偷。如果只有一家，那么根本构不成相邻的条件，就偷唯一的那户人家。
如果有两家，那么偷的必然是两家中钱多的那家。

那如果总数大于 2 家呢？

对于第 n 家来说，只有两种选择:偷或者不偷。

是咋么计算出当前这家是否要偷的呢。

我们假设当前这家编号为 n,那么

max(我偷第 n 家的钱 + sum(截止第 n-2 家偷的钱), sum(截止第 n-1 家偷的钱，因为既然 n-1 家偷了，那么 n 必然是不能偷了))。

这才是这道题关于动态规划最核心的一个点。

看懂了吗？没看懂也没关系，手把手摸个图片出来。
![image](https://image.syst.top/image/leetcode/213-1.png)

还没看懂，加我微信 `remember_wuqinqiang` 我告诉你。这里顺便打个广告，没加我好友的赶紧加我好友。


最后，还需要考虑一个问题，如何确保偷了第一家就不偷最后一家，偷最后一家就不偷第一家。
很简单，直接定义两个dp，一个范围不包括第一家，一个范围不包括最后一家。
最后我们变成了求:
```go
// 伪代码
最佳偷钱:=max(dp[不包括第一家],dp[不包括最后一家])
```
那么剩下的就是对两个dp的状态转移公式了。
```go
func rob(nums []int) int {
	n := len(nums)
	if n == 0 {
		return 0
	}
	if n == 1 {
		return nums[0]
	}

	dp1, dp2 := make([]int, n), make([]int, n)

	dp1[0] = nums[0]
	dp1[1] = max(dp1[0], nums[1])

	dp2[0] = 0 //dp2 不算第一家了
	dp2[1] = max(dp2[0], nums[1])

	for i := 2; i < n; i++ {
		if i < n-1 { // dp1 不算最后一家
			dp1[i] = max(dp1[i-1], dp1[i-2]+nums[i])
		} else {
			dp1[i] = dp1[i-1]
		}
		dp2[i] = max(dp2[i-1], dp2[i-2]+nums[i])
	}
	return max(dp2[n-1], dp1[n-1])
}

func max(x int, y int) int {
	if x > y {
		return x
	}
	return y
}

```
其实代码还能更简洁。

```go
func rob(nums []int) int {
	n := len(nums)
	if n == 0 {
		return 0
	}

	if n == 1 {
		return nums[0]
	}
	if n == 2 {
		return max(nums[0], nums[1])
	}
	return max(helper(nums[1:]), helper(nums[:n-1]))
}

func helper(nums []int) int {
	first, second := nums[0], max(nums[0], nums[1])
	for _, v := range nums[2:] {
		first, second = second, max(second, first+v)
	}
	return second
}

func max(a, b int) int {
	if a > b {
		return a
	}
	return b
}
```




