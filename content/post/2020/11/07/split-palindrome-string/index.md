---
title: "131.分割回文串"
description: ""
date: 2020-11-07T15:15:03+08:00
draft: false
tags: 
  - 算法
  - leetcode
categories: 
  - 算法
slug: "split-palindrome-string"
---

### 一、题目

![题目](question.PNG)

### 二、解法

#### 回溯+动态规划优化

搜索问题主要使用回溯法。

回溯法思考的步骤：

1、画递归树；

2、根据自己画的递归树编码。

![递归树](E:\workspace\github-page\blog\content\post\2020\11\07\split-palindrome-string\recursive-tree.png)

思考如何根据这棵递归树编码：

1、每一个结点表示剩余没有扫描到的字符串，产生分支是截取了剩余字符串的前缀；

2、产生前缀字符串的时候，判断前缀字符串是否是回文。

如果前缀字符串是回文，则可以产生分支和结点；
如果前缀字符串不是回文，则不产生分支和结点，这一步是剪枝操作。
3、在叶子结点是空字符串的时候结算，此时**从根结点到叶子结点的路径，就是结果集里的一个结果，使用深度优先遍历，记录下所有可能的结果**。

- 采用一个路径变量 `path` 搜索，`path` 全局使用一个（注意结算的时候，需要生成一个拷贝），因此在递归执行方法结束以后需要回溯，即将递归之前添加进来的元素拿出去；
- `path` 的操作只在列表的末端，因此合适的数据结构是栈。

利用「力扣」第 5 题：[最长回文子串](https://leetcode-cn.com/problems/longest-palindromic-substring) 的思路，使用空间换时间，利用动态规划把结果先算出来，这样就可以以 **O(1)** 的时间复杂度直接得到一个子串是否是回文。

### 三、代码

```go
func partition(s string) [][]string {
ans := make([][]string,0)
	size := len(s)
	if size == 0 {
		return ans
	}
	// 预处理
	// 使用 dp[i][j] 表示 s[i:j] 是否回文
	dp := make([][]bool, size)
	for i := range dp {
		dp[i] = make([]bool, size)
	}
	for r:=0; r<size; r++ {
		for l:=0; l<=r; l++ {
			if s[l] == s[r] && (r - l <=2 || dp[l+1][r-1]) {
				dp[l][r] = true
			}
		}
	}

	path := make([]string, 0)

	var backtracking func(start int)
	backtracking = func (start int) {
		if start >= size {
			tmp := make([]string, len(path))
			copy(tmp, path)
			ans = append(ans, tmp)
			return
		}
		for i := start; i < size; i++ {
			if !dp[start][i] {
				continue
			}
			path = append(path, s[start:i+1])
			backtracking(i+1)
			path = path[:len(path)-1]
		}
	}
	backtracking(0)
	return ans
}
```



### 四、参考

**转载自：**[回溯、优化（使用动态规划预处理数组） - 分割回文串 - 力扣（LeetCode） (leetcode-cn.com)](https://leetcode-cn.com/problems/palindrome-partitioning/solution/hui-su-you-hua-jia-liao-dong-tai-gui-hua-by-liweiw/)