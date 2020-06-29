---
layout: post
title:  使用goroutine channel顺序输出foobarfoobar
date:   2020-06-28 22:46:00 +0800
categories: golang
tags: goroutine同步,channel
---

题目：改造如下代码使程序输出foobarfoobar

```
type FooBar struct {
	cnt int
}

func (fb *FooBar) foo() {
	for i := 0; i < fb.cnt; i++ {
		fmt.Printf("foo")
	}
}

func (fb *FooBar) bar() {
	for i := 0; i < fb.cnt; i++ {
		fmt.Printf("bar")
	}
}
```

实现代码：

```
package main

import (
	"fmt"
	"time"
	"runtime"
)

type FooBar struct {
	cnt int
	ch1 chan struct{}
	ch2 chan struct{}
}

func (fb *FooBar) foo() {
	for i := 0; i < fb.cnt; i++ {
		fmt.Printf("foo")
		fb.ch1 <- struct{}{}
		<-fb.ch2
	}
}

func (fb *FooBar) bar() {
	for i := 0; i < fb.cnt; i++ {
		fmt.Printf("bar")
		<-fb.ch1
		fb.ch2 <- struct{}{}
	}
}

func main() {
	// 必须设置成1，保证同一时刻只有一个执行中的线程
	runtime.GOMAXPROCS(1)
	fb := &FooBar{cnt: 2, ch1: make(chan struct{}), ch2: make(chan struct{})}
	go fb.foo()
	go fb.bar()
	time.Sleep(1 * time.Second)
}
```

注意：程序中必须设置runtime.GOMAXPROCS(1)，保证同一时刻只有一个执行中的线程。
