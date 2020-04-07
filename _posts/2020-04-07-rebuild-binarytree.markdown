---
layout: post
title:  重建二叉树
date:   2020-04-07 16:35:00 +0800
categories: algorithms
tags: 二叉树,二叉树遍历,golang,算法
---

### 题目描述  

输入某二叉树的前序遍历和中序遍历的结果，请重建出该二叉树。假设输入的前序遍历和中序遍历的结果中都不含重复的数字。例如输入前序遍历序列{1,2,4,7,3,5,6,8}和中序遍历序列{4,7,2,1,5,3,8,6}，则重建二叉树并返回。

### 解题思路

二叉树规则：已知前/后序列 和 中序序列可唯一确定一个二叉树。

大概思路：

取前序序列的第1个元素e作为二叉树的根结点，以元素e为中心分割中序序列得到leftInOrders和rightInOrders，将e的左子树指向leftInOrders的根结点，e的右子树指向rightInOrders的根结点；依次递归执行，先构造左子树再构造右子树。

具体步骤：

1. 前序序列第1个元素“1”作为二叉树根结点；然后拆分中序序列，{4,7,2}是“1”的左子树，{5,3,8,6}是“1”右子树；
2. 前序序列的第2个元素值是“2”作为左子树{4,7,2}的根结点；然后拆分中序序列，{4,7}为“2”的左子树，“2”无右子树；
3. 前序序列的第3个元素值是“4”作为{4,7}的根结点；然后拆分中序序列，{7}是为“4”的右子树，“4“没有左子树；
4. 前序序列的第4个元素值是“7”作为{7}的根结点；至此左子树构建完毕，之后开始构建右子树。
5. 前序序列的第5个元素值是“3”作为右子树序列{5,3,8,6}的根结点；然后拆分中序序列，{"5"}是“3”的左子树，{“8”,“6”}是“3”的右子树；
6. 前序序列的第6个元素值是“5”作为{"5"}的根结点；空序列返回。
7. 前序序列的第7个元素值是”6“作为{"8",“6”}的根结点；然后拆分中序序列，{"8"}是"6"的左子树，“6”没有右子树
8. 前序序列的第8个元素值是“8”作为{"8"}的根结点；空序列返回。



### 代码实现

{% highlight golang %}

// BTree : 二叉树结构体
type BTree struct {
	elem           int
	lchild, rchild *BTree
}

// RebuildBinaryTreeByPreInOrders : 根据前序/中序序列生成二叉树
func RebuildBinaryTreeByPreInOrders(preOrders []int,
	inOrders []int, pos *int) *BTree {
	if len(preOrders)-1 < *pos || len(inOrders) == 0 {
		return nil
	}

	// 获取根结点
	root := preOrders[*pos]
	p := &BTree{elem: root}
	(*pos)++

	// 根据根结点分割中序序列
	leftInOrders, rightInOrders := splitInOrders(
		inOrders, root)
	if len(leftInOrders) > 0 {
		p.lchild = RebuildBinaryTreeByPreInOrders(
			preOrders, leftInOrders, pos)
	}
	if len(rightInOrders) > 0 {
		p.rchild = RebuildBinaryTreeByPreInOrders(
			preOrders, rightInOrders, pos)
	}

	return p
}

// 分割中序序列
func splitInOrders(inOrders []int, root int) ([]int, []int) {
	if len(inOrders) == 0 {
		return nil, nil
	}
	// root值在inOrders序列中的位置
	pos := 0
	for i, v := range inOrders {
		if root == v {
			pos = i
			break
		}
	}
	return inOrders[:pos], inOrders[pos+1:]
}

// VisitFunc : 遍历函数
type VisitFunc func(node *BTree)

// NullFunc : 空函数，什么都不做
func NullFunc(node *BTree) {

}

// InOrder : 中序遍历
func (b *BTree) InOrder(f VisitFunc, list *[]int) {
	if b == nil {
		return
	}

	b.lchild.InOrder(f, list)
	f(b)
	*list = append(*list, b.elem)
	b.rchild.InOrder(f, list)

}

// PreOrder : 前序遍历
func (b *BTree) PreOrder(f VisitFunc, list *[]int) {
	if b == nil {
		return
	}
	f(b)
	*list = append(*list, b.elem)
	b.lchild.PreOrder(f, list)
	b.rchild.PreOrder(f, list)
}

{% endhighlight %}

测试用例：

{% highlight golang %}
func TestRebuildTree(t *testing.T) {
	preOrders := []int{1, 2, 4, 7, 3, 5, 6, 8}
	inOrders := []int{4, 7, 2, 1, 5, 3, 8, 6}
	var preOrderPos int = 0

	root := RebuildBinaryTreeByPreInOrders(
		preOrders, inOrders, &preOrderPos)

	inList := []int{}
	root.InOrder(NullFunc, &inList)

	preList := []int{}
	root.PreOrder(NullFunc, &preList)

	t.Log(inList, preList)

	if !reflect.DeepEqual(inOrders, inList) {
		t.Errorf("expected %v, but has %v",
			inOrders, inList)
	}

	if !reflect.DeepEqual(preOrders, preList) {
		t.Errorf("expected %v, but has %v",
			preOrders, preList)
	}
}
{% endhighlight %}