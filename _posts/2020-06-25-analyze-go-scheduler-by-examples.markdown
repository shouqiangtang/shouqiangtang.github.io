---
layout: post
title:  实例分析go调度原理
date:   2020-06-25 11:41:00 +0800
categories: golang
tags: go调度原理,go互斥锁
---


下面根据两个实例来分析下[Go调度器](https://morsmachine.dk/go-scheduler)的调度机制及go互斥锁的原理。

本机环境：

```
➜  ~ go version
go version go1.13.5 darwin/amd64
➜  ~ uname -a
Darwin *** 19.4.0 Darwin Kernel Version 19.4.0: Wed Mar  4 22:28:40 PST 2020; ***
```

# 实例1

```
package main

import (
	"fmt"
	"sync"
)

const N = 10

func main() {
	wg := sync.WaitGroup{}

	wg.Add(N)

	for i := 0; i < N; i++ {
		go func() {
			defer wg.Done()
			fmt.Printf("%d ", i)
		}()
	}

	wg.Wait()
}
```

**罗列了两个可能的输出结果，如下：**

```
3 10 10 10 10 10 10 10 10 10
10 10 10 10 10 10 10 10 10 10
```

**总结分析：**

1. 函数体中的变量i和for循环中的i是同一变量，即循环中的i和函数体中的i使用的同一段内存。因此不会输出1 2 3 4 5 6 7 8 9 10，可以如下修改代码：
```
    ...
    
    for i := 0; i < N; i++ {
		go func(i int) {
			defer wg.Done()
			fmt.Printf("%d ", i)
		}(i)
	}
    
    ...
```

2. main程(执行for循环)和其它go程(输出变量i)，这些go程何时运行依赖于Go调度器，main程执行中，其它go程有可能被调度，但具有不确定性。比如从第一条输出结果上看，在main程执行中，调度器激活了某个go程输出了3.

# 实例2

```
package main

import (
	"fmt"
	"sync"
)

const N = 20

func main() {
	wg := sync.WaitGroup{}
	mu := sync.Mutex{}
	m := make(map[int]int)
    debug := make([]int, N)

	wg.Add(N)
	for i := 0; i < N; i++ {
		go func(j int) {
			mu.Lock()
			defer wg.Done()
			// fmt.Println(j, i)
			debug[j] = i
			m[i] = i
			mu.Unlock()
		}(i)
	}
	wg.Wait()

	for k, v := range debug {
		fmt.Printf("第%d个go程，i=%d\n", k, v)
	}
	fmt.Println(len(m), m)
}

```

**程序输出具有不确定性，可能的输出结果如下：**

```
第0个go程，i=4
第1个go程，i=12
第2个go程，i=12
第3个go程，i=12
第4个go程，i=12
第5个go程，i=11
第6个go程，i=12
第7个go程，i=12
第8个go程，i=12
第9个go程，i=12
第10个go程，i=12
第11个go程，i=12
第12个go程，i=15
第13个go程，i=17
第14个go程，i=17
第15个go程，i=18
第16个go程，i=19
第17个go程，i=20
第18个go程，i=20
第19个go程，i=20
8 map[7:11 11:11 12:12 15:17 17:17 18:18 19:19 20:20]
```

**知识点：**

* 如果map不加锁时会报"fatal error: concurrent map writes"
* fmt.Println是I/O操作会触发协程调度
* 某个go程获得锁后，其它go程抢锁时会堵塞等待。

**总结分析：**

* i值相同的go程在等待锁。反过来根据现象推，go程等待锁时变量i的值已经确定了，即便是m[i]=i语句在mu.Lock()语句之后执行。也就是说等待锁的go程会保存上下文(比如i的值)并等待被Go调度器重新调度；，
* 等待锁的go程被优先调度的机率要比其它go程要大很多
