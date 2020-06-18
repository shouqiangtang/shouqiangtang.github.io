---
layout: post
title:  快速排序
date:   2020-04-12 21:35:00 +0800
categories: algorithms
tags: 快速排序,golang,算法
---

### 算法描述  

通常，快速排序被认为是在所有同数量级（O(nlogn)）的排序方法中，其平均性能最好。

它的基本思想是，通过一趟排序将待排记录分割成独立的两部分，其中一部分的关键字均比另一部分的关键字小，以此类推分别对这两部分继续排序，直到整个记录有序。

首先人意选取一个记录（通常可选第一个记录）作为枢轴（pivot），然后按下述原则重新排列其余记录：将所有关键字比它小的记录都安置在它的位置之前，将所有关键字比它大的记录都安置在它的位置之后。由此可以将“枢轴”记录最后所落的位置i作分界线，分割成两个子序列。这个过程称做一趟分区。


### 代码描述

```
package quicksort

// 4, 6, 7, 8, 3, 1, 2, 5 -> 4
// pivot = arr[0] = 4, index = 1
// i, arr[i] < pivot,      index,   arr
// 1  6 < 4 == false       1        (4,6,7,8,3,1,2,5)
// 2  7 < 4 == false       1        (4,6,7,8,3,1,2,5)
// 3  8 < 4 == false       1        (4,6,7,8,3,1,2,5)
// 4  3 < 4 == true        2        (4,3,7,8,6,1,2,5)
// 5  1 < 4 == true        3        (4,3,1,8,6,7,2,5)
// 6  2 < 4 == true        4        (4,3,1,2,6,7,8,5)
// 7  5 < 4 == false       4        (4,3,1,2,6,7,8,5)
// swap(arr, 0, index-1) => (4,3,1,2,6,7,8,5) -> (2,3,1,4,6,7,8,5)
//
// 分区算法1
// 算法描述：该算法关键是定义一个index变量，index指向第一个大于pivot的位置或者指向
// 数组的末尾（数组中pivot最大）
func partition(arr []int, left, right int) int {
	pivot := arr[left] // 设定基准值（pivot）
	index := left + 1
	for i := index; i <= right; i++ {
		if arr[i] < pivot {
			arr[i], arr[index] = arr[index], arr[i]
			index++
		}
	}
	arr[left], arr[index-1] = arr[index-1], arr[left]
	return index - 1
}

// partition : 分区算法2
// 此方法是《数据结构》书上的算法。
// 1.将右指针所指值与pivot比较，如果右指针(right)所指关键字大于等于pivot，则右指针左移(right--)；否则将右指针的值赋给左指针
// 2.将左指针所指值与pivot比较，如果左指针(left)所指关键字小于等于pivot，则左指针右移(left++)；否则将左指针的值赋给右指针
// 3.重复执行1，2直到left大于等于right
// 4.最后将pivot值赋给left
func partition2(arr []int, left, right int) int {
	pivot := arr[left] // 设定基准值（pivot）
	for left < right {
		for left < right && arr[right] >= pivot {
			right--
		}
		arr[left] = arr[right]
		for left < right && arr[left] <= pivot {
			left++
		}
		arr[right] = arr[left]
	}
	arr[left] = pivot
	return left
}

// QuickSort : 快速排序
func QuickSort(arr []int, left, right int) {
	if left >= right {
		return
	}
	pivot := partition2(arr, left, right)
	QuickSort(arr, left, pivot-1)
	QuickSort(arr, pivot+1, right)
}
```
