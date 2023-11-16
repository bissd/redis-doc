---
title: "Redis CPU 分析"
linkTitle: "CPU性能分析"
weight: 1
description: >
    在CPU性能工程指南中对于性能分析和跟踪的指导
aliases: [
    /topics/performance-on-cpu,
    /docs/reference/optimization/cpu-profiling
]
---

## 填写性能清单

Redis以出色的性能为基础开发。我们在每个版本中尽力确保您能获得稳定而快速的产品。

然而，如果您正在寻找改进 Redis 效率的空间，或者正在进行性能退化调查，您需要一种简洁、系统性的监视和分析 Redis 性能的方法。

为此，您可以依赖不同的方法（根据我们打算进行的问题/分析类别，有些方法更适合）。Brendan Greg在以下链接中列出了一份精选的方法和步骤清单：[链接](http://www.brendangregg.com/methodology.html)。

我们推荐使用Utilization Saturation and Errors (USE) 方法来回答什么是你的瓶颈问题。请查看以下系统资源、指标和工具之间的实用深入分析映射：
[USE method](http://www.brendangregg.com/USEmethod/use-rosetta.html)。

### 确保CPU是瓶颈

这个指南假设您已经按照上述方法之一对系统健康进行了完整的检查，并确定瓶颈是CPU。**如果您确定大部分时间都花在I/O、锁、定时器、分页/交换等方面被阻塞，那么这个指南不适用于您**。

### 构建前提条件

为了进行正确的 On-CPU 分析，Redis（以及任何动态加载的库，如 Redis 模块）需要堆栈跟踪对追踪器可用，你可能首先需要修复它们。

默认情况下，Redis是使用`-O2`开关编译的（我们打算在分析时保留）。这意味着启用了编译器优化。许多编译器在运行时会省略帧指针作为优化（节省一个寄存器），从而导致基于帧指针的堆栈遍历出错。这使得Redis可执行文件运行速度更快，但同时也使得Redis（和其他程序一样）更难追踪，可能错误地将CPU时间定位到最后一个可用帧指针的调用堆栈中，而该堆栈可能更深（但无法追踪）。

非常重要的是确保：
- 调试信息存在：编译选项 `-g`
- 帧指针寄存器存在：`-fno-omit-frame-pointer`
- 我们仍然使用优化以获得准确的生产运行时间表示，这意味着我们将保持：`-O2`

您可以在redis主存储库中按照以下方式进行操作：

    $ make REDIS_CFLAGS="-g -fno-omit-frame-pointer"

## 一组工具，用于识别性能回归和/或潜在的 **CPU 上性能** 改进

这份文档专注于分析**在CPU上**的资源瓶颈，也就是说我们感兴趣的是了解线程在CPU上运行时花费的CPU周期，并且同样重要的是，这些周期是否有效地用于计算，或者是否因为等待（而不是阻塞！）内存I/O、缓存未命中等而处于停滞状态。

为此，我们将依靠工具包（perf、bcc工具）和硬件特定的PMC（性能监控计数器）来继续进行：

- 热点分析（perf 或 bcc 工具）：用于分析代码执行和确定哪些函数占用了最多的时间，因此是优化的目标。我们将介绍两种选项来收集、报告和可视化热点，可以使用 perf 或 bcc/BPF 跟踪工具。

- 调用计数分析：用于计算包括函数调用在内的事件，使我们能够一次关联多个调用/组件，依靠bcc/BPF跟踪工具。

- 硬件事件采样：对于理解CPU行为至关重要，包括内存I/O、停顿周期和缓存未命中。

### 工具前提条件

以下步骤依赖于Linux perf_events（又称 ["perf"](https://man7.org/linux/man-pages/man1/perf.1.html)）、[bcc/BPF跟踪工具](https://github.com/iovisor/bcc)和Brendan Greg的[FlameGraph仓库](https://github.com/brendangregg/FlameGraph)。

我们假设您事先具备以下条件：

- 在您的系统上安装了perf工具。大多数Linux发行版可能会将其打包为与内核相关的软件包。有关perf工具的更多信息，请访问perf [wiki](https://perf.wiki.kernel.org/)。
- 按照安装[bcc/BPF](https://github.com/iovisor/bcc/blob/master/INSTALL.md#installing-bcc)说明，在您的计算机上安装了bcc工具包。
- 克隆了Brendan Greg的[FlameGraph仓库](https://github.com/brendangregg/FlameGraph)，并使`difffolded.pl`和`flamegraph.pl`文件可访问，以生成折叠的堆栈跟踪和火焰图。

使用perf或eBPF（堆栈跟踪抽样）进行热点分析

# 通过定时间隔采样堆栈跟踪来分析CPU使用率是一种快速简便的方式，用于识别性能关键代码段（热点）。

### 使用 perf 采样堆栈跟踪

要对redis-server的用户级和内核级堆栈进行配置文件，以特定的时间长度进行配置文件，例如60秒，采样频率为每秒999个样本：

    $ perf record -g --pid $(pgrep redis-server) -F 999 -- sleep 60

使用perf报告显示记录的配置文件信息

默认情况下，perf record 会在当前工作目录生成一个 perf.data 文件。

你可以使用调用图输出（调用链、堆栈回溯）进行报告，
调用图最低包含阈值为0.5%，具体操作如下：

    $ perf report -g "graph,0.5,caller"

请参阅[perf报告](https://man7.org/linux/man-pages/man1/perf-report.1.html) 的文档，了解高级过滤、排序和聚合功能。

#### 使用火焰图可视化记录的性能分析信息

[火焰图](http://www.brendangregg.com/flamegraphs.html)允许快速准确地可视化频繁的代码路径。它们可以使用Brendan Greg在[github](https://github.com/brendangregg/FlameGraph)上的开源程序生成，该程序从折叠的堆栈文件创建交互式SVG。

具体来说，我们需要将生成的perf.data转换为捕获的堆栈，并将每个堆栈折叠为单行。然后，您可以使用以下命令渲染CPU热点图：

    $ perf script > redis.perf.stacks
    $ stackcollapse-perf.pl redis.perf.stacks > redis.folded.stacks
    $ flamegraph.pl redis.folded.stacks > redis.svg

默认情况下，perf脚本将在当前工作目录中生成一个名为perf.data的文件。有关高级用法，请参阅[perf脚本](https://linux.die.net/man/1/perf-script)文档。

请查看[FlameGraph使用选项](https://github.com/brendangregg/FlameGraph#options)获取更高级的堆栈跟踪可视化选项（如差异化）。

#### 归档和共享记录的个人资料信息

为了使非收集数据发生的计算机上能够对perf.data内容进行分析，您需要将与记录数据文件中发现的构建ID相关的所有目标文件与perf.data文件一起导出。这可以通过`perf-archive.sh`脚本轻松完成：（[perf-archive.sh](https://github.com/torvalds/linux/blob/master/tools/perf/perf-archive.sh)）

    $ perf-archive.sh perf.data

现在请运行：

    $ tar xvf perf.data.tar.bz2 -C ~/.debug

在需要运行`perf report`的机器上。

### 使用bcc/BPF的profile功能进行采样堆栈跟踪
    
与perf类似，自Linux内核4.9版本开始，BPF优化的性能分析功能已完全可用。它承诺在CPU（通过在内核上下文中频度计数堆栈跟踪）和磁盘I/O资源方面带来更低的开销。

除此之外，仅仅依靠 bcc/BPF 的配置工具，我们还移除了 perf.data 和中间步骤，以便于堆栈跟踪分析成为主要目标。您可以使用 bcc 的配置工具直接输出折叠格式，用于生成火焰图。

    $ /usr/share/bcc/tools/profile -F 999 -f --pid $(pgrep redis-server) --duration 60 > redis.folded.stacks

按这种方法，我们已经移除了任何预处理，并且可以用一个命令渲染在 CPU 上的火焰图：

    $ flamegraph.pl redis.folded.stacks > redis.svg

### 使用火焰图可视化记录的性能分析信息

使用bcc/BPF进行调用次数分析

函数可能会消耗大量的CPU周期，原因是其代码较慢或者其被频繁调用。要回答函数被调用的频率，您可以使用BCC的`funccount`工具进行调用计数分析：

    $ /usr/share/bcc/tools/funccount 'redis-server:(call*|*Read*|*Write*)' --pid $(pgrep redis-server) --duration 60
    Tracing 64 functions for "redis-server:(call*|*Read*|*Write*)"... Hit Ctrl-C to end.

    FUNC                                    COUNT
    call                                      334
    handleClientsWithPendingWrites            388
    clientInstallWriteHandler                 388
    postponeClientRead                        514
    handleClientsWithPendingReadsUsingThreads      735
    handleClientsWithPendingWritesUsingThreads      735
    prepareClientToWrite                     1442
    Detaching...


上面的输出显示，在跟踪过程中，Redis的call()函数被调用了334次，handleClientsWithPendingWrites()被调用了388次，等等。

## 使用性能监测计数器（PMCs）进行硬件事件计数

许多现代处理器都包含性能监测单元（PMU），可以公开性能监测计数器（PMC）。 PMC对于理解CPU行为至关重要，包括内存I/O，停顿周期和缓存失效，并提供了其他任何地方都不可用的低级CPU性能统计信息。

PMU的设计和功能是CPU具体的，您应该使用`perf list`来评估CPU支持的计数器和特性。

为了计算每个周期的指令数、执行的微操作数、不调度微操作的周期数，以及内存阻塞周期数，包括每种类型的内存阻塞，在60秒的持续时间内，特别针对Redis进程进行如下操作：

    $ perf stat -e "cpu-clock,cpu-cycles,instructions,uops_executed.core,uops_executed.stall_cycles,cache-references,cache-misses,cycle_activity.stalls_total,cycle_activity.stalls_mem_any,cycle_activity.stalls_l3_miss,cycle_activity.stalls_l2_miss,cycle_activity.stalls_l1d_miss" --pid $(pgrep redis-server) -- sleep 60

    Performance counter stats for process id '3038':

      60046.411437      cpu-clock (msec)          #    1.001 CPUs utilized          
      168991975443      cpu-cycles                #    2.814 GHz                      (36.40%)
      388248178431      instructions              #    2.30  insn per cycle           (45.50%)
      443134227322      uops_executed.core        # 7379.862 M/sec                    (45.51%)
       30317116399      uops_executed.stall_cycles #  504.895 M/sec                    (45.51%)
         670821512      cache-references          #   11.172 M/sec                    (45.52%)
          23727619      cache-misses              #    3.537 % of all cache refs      (45.43%)
       30278479141      cycle_activity.stalls_total #  504.251 M/sec                    (36.33%)
       19981138777      cycle_activity.stalls_mem_any #  332.762 M/sec                    (36.33%)
         725708324      cycle_activity.stalls_l3_miss #   12.086 M/sec                    (36.33%)
        8487905659      cycle_activity.stalls_l2_miss #  141.356 M/sec                    (36.32%)
       10011909368      cycle_activity.stalls_l1d_miss #  166.736 M/sec                    (36.31%)

      60.002765665 seconds time elapsed

重要的是要知道有两种非常不同的方式可以使用 PMCs（计数和采样），而我们在这个分析中仅仅专注于 PMCs 的计数。Brendan Greg 在下面的链接中清楚地解释了这一点：[link](http://www.brendangregg.com/blog/2017-05-04/the-pmcs-of-ec2.html)。

