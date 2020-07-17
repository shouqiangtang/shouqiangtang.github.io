---
layout: post
title: Go调度器：MS，PS & GS
date: 2020-07-15 22:01:00 +0800
categories: Go
tags: Go调度器,Go调度原理
---

翻译：https://povilasv.me/go-scheduler/?nsukey=pLCzEVslB7vaNKJkTihwvDPGxHGeA0Jwrk7mvAjRTdVSzg1jEwLuAKDWi8nSMcJ7CmjJa6j35IWRf0Q8QgR1%2FpOaWHvX8JyTvVkOOdJthYZt0K6MTCPmcCk4ap9OQJ80%2BexkeSz3y7dyXHmOuHOL4K0Pd%2F4bcmbzBdn%2F2%2BgfIcARuMjh80lV0dEaQ00KEW2PTDtcqyTm0iadJ3CsgsqHTA%3D%3D#

我一直在学习Go，并刚读完《The Go Programming Language》 和 《Go in Action books》两本书，因此我决定学习更多Go内部结构知识。由于有人撰写Go调度器已经很长时间了，所以我认为这将是一个有趣的文章。让我们开始吧！

# 基本概念(Basics)

Go runtime管理调度、垃圾回收和goroutine运行时环境。在这里，我只介绍调度器。

Runtime Scheduler会映射goroutines到操作系统线程。Goroutine是轻量级线程并且启动成本非常低。每个Goroutine由G结构体表示，该结构体包含跟踪其堆栈和当前状态所必需的字段。因此，G=goroutine。

Runtime跟踪每个G并将其映射到逻辑处理器P。P可被视为必需的抽象资源或上下文，以便OS线程（称为M或Machine）可以执行G。

通过调用runtime.GOMAXPROCS(numLogicalProcessors)来控制运行时的逻辑处理器P的数量，如果您打算调整此参数（您可能不应该），则只需设置一次，然后忘记它，因为它需要"Stop The World" GC停顿。

本质上，操作系统运行线程，线程运行你的代码。Go的诀窍是在各个位置（比如，通过channel发送值，调用runtime包）将调用插入到Go runtime，以便Go可以通知调度程序并采取措施。注意：下图的想法来自[Analysis of the Go runtime scheduler](http://www.cs.columbia.edu/~aho/cs6998/reports/12-12-11_DeshpandeSponslerWeiss_GO.pdf)。

![Go Executable](/assets/go-executable.png)

# Ms, Ps & Gs交互（The dance between Ms, Ps & Gs）

Ms，Ps和Gs之间的交互有点复杂。 看看这个惊人的工作流程图，它来自[go runtime scheduler slides by Gao Chao](https://speakerdeck.com/retervision/go-runtime-scheduler)。

![go scheduler workflow](/assets/go-scheduler-workflow.png)

在这里，我们可以看到G的队列有两种类型：[schedt结构体](https://github.com/golang/go/blob/5dd978a283ca445f8b5f255773b3904497365b61/src/runtime/runtime2.go#L536)中的全局队列（很少使用），和每个P维护一个可运行G的队列。

为了执行Goroutine，M必须保持一个上下文P。然后，M从它的P的Goroutine队列中弹出G并执行代码。

当你调度一个新的goroutine（执行go func()调用）时，该goroutine会放入P的运行队列中。有一种有趣的窃取工作调度算法，它运行在当M完成执行某个G并尝试从空队列取另一个G时，然后随机选择另一个P并尝试窃取其可运行G的一半。

当你的goroutine执行一个堵塞的系统调用时会发生有趣的事情。阻塞的系统调用会被拦截，如果存在Gs要运行，runtime会将P和内核线程脱离并创建一个新的线程（如果不存在idle线程的话）来服务P。

如果goroutine执行一个网络调用，runtime将会执行相似的行为。网络调用也会被拦截，但是由于Go具有集成的[网络轮询器network poller](https://morsmachine.dk/netpoller)（具有自己的线程），因此该网络调用将会分配给它。

实际上Go运行时将会运行一个不同的goroutine，如果当前goroutine被堵塞：

* 堵塞的系统调用(比如打开一个文件)
* 网络输入Network input
* channel操作
* sync包原语

# 调度器跟踪

Go允许跟踪运行时调度器。通过GODEBUG环境变量可以做到：

```
GODEBUG=scheddetail=1,schedtrace=1000 ./program
```

这是一个输出实例：

```
SCHED 0ms: gomaxprocs=8 idleprocs=7 threads=2 spinningthreads=0 idlethreads=0 runqueue=0 gcwaiting=0 nmidlelocked=0 stopwait=0 sysmonwait=0
P0: status=1 schedtick=0 syscalltick=0 m=0 runqsize=0 gfreecnt=0
P1: status=0 schedtick=0 syscalltick=0 m=-1 runqsize=0 gfreecnt=0
P2: status=0 schedtick=0 syscalltick=0 m=-1 runqsize=0 gfreecnt=0
P3: status=0 schedtick=0 syscalltick=0 m=-1 runqsize=0 gfreecnt=0
P4: status=0 schedtick=0 syscalltick=0 m=-1 runqsize=0 gfreecnt=0
P5: status=0 schedtick=0 syscalltick=0 m=-1 runqsize=0 gfreecnt=0
P6: status=0 schedtick=0 syscalltick=0 m=-1 runqsize=0 gfreecnt=0
P7: status=0 schedtick=0 syscalltick=0 m=-1 runqsize=0 gfreecnt=0
M1: p=-1 curg=-1 mallocing=0 throwing=0 preemptoff= locks=1 dying=0 helpgc=0 spinning=false blocked=false lockedg=-1
M0: p=0 curg=1 mallocing=0 throwing=0 preemptoff= locks=1 dying=0 helpgc=0 spinning=false blocked=false lockedg=1
G1: status=8() m=0 lockedm=0
```

清注意它使用相同的概念G，M和P及它们的状态，例如P的队列大小。通常，您不需要那么多的细节，因此您可以使用：

```
GODEBUG=schedtrace=1000 ./program
```

William Kennedy写了一片很棒的文章，它介绍了如何阐释这些内容以及跟踪的详细版本。

此外，还有一个名为go tool trace的高级工具，通过该工具的UI可以探索您的程序和运行时正在做什么。您可以在[Pusher文章](https://making.pusher.com/go-tool-trace)中学习如何使用它。


参考资料：

[Slides by Matthew Dale](https://www.slideshare.net/matthewrdale/demystifying-the-go-scheduler)  
[Columbia University paper: Analysis of the Go runtime scheduler](http://www.cs.columbia.edu/~aho/cs6998/reports/12-12-11_DeshpandeSponslerWeiss_GO.pdf)  
[Scalable Go Scheduler Design Doc](https://docs.google.com/document/d/1TTj4T2JO42uD5ID9e89oa0sLKhJYD0Y_kqxDv3I3XMw/edit?usp=sharing)  
[Hacker news chat which explains a lot](https://news.ycombinator.com/item?id=12459841)  
[go runtime scheduler slides by Gao Chao](https://speakerdeck.com/retervision/go-runtime-scheduler)  
[Morsmachine article](http://morsmachine.dk/go-scheduler)  

