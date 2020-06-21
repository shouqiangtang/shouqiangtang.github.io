---
layout: post
title:  八皇后golang版
date:   2020-06-21 22:30:00 +0800
categories: golang
tags: 八皇后
---

# 描述：
八皇后问题是一个古老而著名的问题，是回溯算法的典型例题。该问题是十九世纪著名的数学家高斯1850年提出：在8X8格的国际象棋上摆放八个皇后（棋子），使其不能互相攻击，即任意两个皇后都不能处于同一行、同一列或同一斜线上。

回溯法（back tracking）（探索与回溯法）是一种选优搜索法，又称为试探法，按选优条件向前搜索，以达到目标。但当探索到某一步时，发现原先选择并不优或达不到目标，就退回一步重新选择，这种走不通就退回再走的技术为回溯法，而满足回溯条件的某个状态的点称为“回溯点”


# 代码实现：

```
package main

import (
	"fmt"
	"math"
)

// 记录所有正确的棋子布局
var result [][]int = [][]int{}

func printQueue(queue []int) {
	for _, colpos := range queue {
		for col := 0; col < len(queue); col++ {
			if col == colpos {
				fmt.Printf("1 ")
			} else {
				fmt.Printf("* ")
			}
		}
		fmt.Println()
	}
	fmt.Println("----------------------")
}

// param row 表示行数
// param queue 记录皇后位置
func trial(queue []int, row int) {
	n := len(queue)
	if row == n {
		newQueue := make([]int, n)
		copy(newQueue, queue)
		printQueue(newQueue)
		result = append(result, newQueue)
		return
	}

	for column := 0; column < n; column++ {
		// 落子
		queue[row] = column
		if isValid(queue, row, column) {
			trial(queue, row+1)
		}
	}
}

// 验证皇后位置是否正确
func isValid(queue []int, rowPos, columnPos int) bool {
	for i := 0; i < rowPos; i++ {
		// 判断是否在同一列
		if queue[i] == columnPos {
			return false
		}
		// 是否在对角线上，两个点的x和y方面的距离相等
		if math.Abs(float64(rowPos-i)) == math.Abs(float64(queue[i]-columnPos)) {
			return false
		}
	}
	return true
}

func main() {
	var (
		n int = 8
		queue []int = make([]int, n)
	)
	trial(queue, 0)

	fmt.Println(len(result))
}
```

# 程序输出

```

......

----------------------
* * * * * * * 1 
* * 1 * * * * * 
1 * * * * * * * 
* * * * * 1 * * 
* 1 * * * * * * 
* * * * 1 * * * 
* * * * * * 1 * 
* * * 1 * * * * 
----------------------
* * * * * * * 1 
* * * 1 * * * * 
1 * * * * * * * 
* * 1 * * * * * 
* * * * * 1 * * 
* 1 * * * * * * 
* * * * * * 1 * 
* * * * 1 * * * 
----------------------
92
```

# 参考文档

[小白带你学---回溯算法（Back Tracking)](https://zhuanlan.zhihu.com/p/54275352)
