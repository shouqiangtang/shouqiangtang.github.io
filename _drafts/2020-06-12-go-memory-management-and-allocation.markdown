---
layout: post
title: Go 内存管理和分配
date:   2020-06-12 06:45:00 +0800
categories: golang
tags: go memory
---

这片文章基于Go 1.13。

Go内存管理，从内存的分配到不再使用时该内存的回收是由Go标准库自动完成。尽管开发人员不必处理它，但是Go进行的基础管理已得到了很好的优化，并且充满了有趣的概念。

# 堆分配

内存管理旨在在并发环境中快速运行，并与垃圾回收器继承在一起。让我们从一个简单的示例开始：

```
package main

type smallStruct struct {
   a, b int64
   c, d float64
}

func main() {
   smallAllocation()
}

//go:noinline
func smallAllocation() *smallStruct {
   return &smallStruct{}
}
```

什么是内联inline？  
> [What is inlining?](https://dave.cheney.net/2020/04/25/inlining-optimisations-in-go)  
> Inlining is the act of combining smaller functions into their respective callers. In the early days of computing this optimisation was typically performed by hand. Nowadays inlining is one of a class of fundamental optimisations performed automatically during the compilation process.  

