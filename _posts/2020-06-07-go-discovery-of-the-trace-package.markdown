---
layout: post
title:  GO Trace包探索
date:   2020-06-07 06:45:00 +0800
categories: golang
tags: go trace
---

这篇文章基于Go 1.13。

Go为我们提供了一个工具，用于在运行时开启追踪，并获取程序执行的详细概览。可以在tests中通过标志-trace启用此工具，从pprof进行实时跟踪，也可以通过跟踪包在代码中的任意位置启用该工具。该工具可以更强大，因为你可以使用自己的跟踪增强它。让我们来回顾下它的工作原理。

# 追踪流程

该工具的流程非常简单。每个事件，例如内存分配、垃圾回收器的所有阶段、goroutine的运行和暂停，由go标准库记录并供以后显示。但是，在录制开始之前，Go首先"stops the world"，并对当前goroutine及其状态取快照。

您可以在这篇文章"[Go:How Does Go Stop the World?](https://medium.com/a-journey-with-go/go-how-does-go-stop-the-world-1ffab8bc8846)"”找到更多关于这个阶段的细。

之后，这将使Go能够正确构建每个goroutine的生命周期。流程如下：

![initialization phase before tracing](/assets/initialization_phase_before_tracing.png)

然后，将收集到的事件推送到缓冲区，当达到最大容量时再刷新到完整缓冲区列表。这是流程图：

![Tracing collect events per P](/assets/tracing_push_events_perp.png)

跟踪器现在需要一种将这些跟踪数据dump到输出的方法。为此，当跟踪器开始时Go会派生一个专用于dump的goroutine。这个goroutine将在可用时dump数据，并将停止park goroutine直到有新的追踪数据产生。这是一个它的表示：

![a dedicated goroutine read and dump the traces](/assets/a_dedicated_goroutine_reads_and_dump_the_traces.png)

现在流程已经很清晰，现在让我查看一下已记录的追踪信息。

# 追踪

生成跟踪后，可以通过运行命令go tool trace my-output.out来完成可视化。让我们以一些追踪数据为例：

![Tracing from go tool](/assets/tracing_from_go_tool.png)

大多数追踪数据直观简单。垃圾回收器相关的追踪数据位于蓝色追踪GC下，如图：

![Traces of the garbage collector](/assets/traces_of_the_garbage_collector.png)

快速回顾：

* STW是垃圾回收器里的两个"Stop the World"阶段。在这两个阶段goroutines停止。
* GC(idle)是在没有工作时标记内存的goroutine。
* MARK ASSIST是在分配过程中帮助标记内存的goroutines。
* GXX runtime.bgsweep是内存扫描阶段，一旦垃圾回收器完成。
* GXX runtime.gcBgMarkWorker是专用后台goroutines，用于帮助标记内存

你可以在我的文章"[Go: How Does the Garbage Collector Watch Your Application?](https://medium.com/a-journey-with-go/go-how-does-the-garbage-collector-watch-your-application-dbef99be2c35)"发现更多的关于那些追踪信息的细节。

然而，一些追踪数据不那么容易理解。让我们来回顾一下以加深理解：

* proc start 被调用是在processor关联thread时。它发生在一个开始新的thread时，或当从系统调用中恢复时。

![trace proc start](/assets/trace_proc_start.png)

* proc stop 被调用是在thread和当前processor解除关联时。它发生在thread阻塞在系统调用时，或当tread退出时。

![trace proc stop](/assets/trace_proc_stop.png)

* syscall 被调用是在goroutine执行系统调用时。

![trace syscall](/assets/trace_syscall.png)

* unblock 被调用是在当goroutine从系统调用中解除阻塞时。— 在那种情形下应显示syscall标签，从一个堵塞的channel，等等。

![trace unblock](/assets/trace_unblock.png)

由于Go允许您定义和可视化自己的跟踪以及标准库中的跟踪，因此可以增强跟踪。


# 用户追踪

我们可以定义的跟踪具有两个层次级别：

* 在最高层使用具有开始和结束的task
* 在子层上使用region

这是一个简单的例子：

```
func main() {
	ctx, task := trace.NewTask(context.Background(), "main start")

	var wg sync.WaitGroup
	wg.Add(2)

	go func() {
		defer wg.Done()
		r := trace.StartRegion(ctx, "reading file")
		defer r.End()

		ioutil.ReadFile(`n1.txt`)
	}()

	go func() {
		defer wg.Done()
		r := trace.StartRegion(ctx, "writing file")
		defer r.End()

		ioutil.WriteFile(`n2.txt`, []byte(`42`), 0644)
	}()

	wg.Wait()

	defer task.End()
}
```

那些新跟踪可从工具中直接可视化，通过用户自定义的任务菜单：

![Custom task and regions](/assets/custom_task_and_regions.png)

也可以记录任意日志到task里：

```
ctx, task := trace.NewTask(context.Background(), "main start")
trace.Log(ctx, "category", "I/O file")
trace.Log(ctx, "goroutine", "2")
```

这些日志将在设置task的goroutine下找到：

![Custom logs in the tracing](/assets/custom_logs_in_the_tracing.png)

这些任务可以被嵌入其它任务中，通过派生父任务的上下文等。

但是，由于pprof的缘故，可以在生产环境实时跟踪所有事件，同时在收集它们时也会稍微降低性能。

# 性能影响

一个简单的基准可以帮助理解跟踪的影响。一个使用-trace标记另外一个不使用。以下是带有ioutil.ReadFile（）函数的基准测试的结果，该函数会生成很多事件：

```
name         time/op
ReadFiles-8  48.1µs ± 0%

name         time/op
ReadFiles-8  63.5µs ± 0%  // with tracing
```

在这种情况下，影响约为35％，并且可能因应用而异。但是，某些工具（例如StackDriver）允许在生产中进行连续配置，同时在我们的应用程序上保持少量开销。

原文地址：[Go: Discovery of the Trace Package](https://medium.com/a-journey-with-go/go-discovery-of-the-trace-package-e5a821743c3c)
