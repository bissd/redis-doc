---
title: "Debugging"
linkTitle: "Debugging"
weight: 10
description: >
    Redis服务器进程调试指南
aliases: [
    /topics/debugging,
    /docs/reference/debugging,
    /docs/reference/debugging.md
]
---

Redis开发时重视稳定性。我们在每个版本中都尽力确保您能够使用稳定的产品，没有崩溃。然而，如果您需要调试Redis进程本身，请继续阅读。

当Redis崩溃时，会生成一个详细的报告来说明发生了什么。然而，有时仅仅查看崩溃报告是不够的，也不可能让Redis核心团队独立地重现问题。在这种情况下，我们需要用户能够重现问题的帮助。

本指南展示了如何使用GDB来提供Redis开发人员跟踪错误所需的信息。

## 什么是GDB？

GDB 是 GNU 调试器：一个能够检查另一个程序的内部状态的程序。通常，追踪和修复一个错误是一个收集有关程序在错误发生时的状态的更多信息的过程，因此 GDB 是一个非常有用的工具。

GDB可以有两种使用方式：

* 它可以附加到正在运行的程序并在运行时检查其状态。
* 它可以使用所谓的“核心文件”检查已经终止的程序的状态，即程序运行时的内存图像。

从调查Redis错误的角度来看，我们需要使用这两种GDB模式。用户可以将GDB附加到正在运行的Redis实例上，在发生崩溃时，他们会创建`core`文件，然后开发者将使用该文件来检查崩溃时的Redis内部状态。

这样开发人员可以在自己的计算机上执行所有的检查工作，
不需要用户的帮助，而用户则可以在生产环境中重启Redis。

# 在不进行优化的情况下编译Redis

默认情况下，Redis是使用"-O2"开关进行编译的，这意味着启用了编译器优化。这使得Redis可执行文件更快，但同时也使得Redis（如同任何其他程序）更难以使用GDB进行检查。

最好是使用`make noopt`命令将Redis编译为未优化版本后再附加GDB，而不是仅使用普通的`make`命令。然而，如果您已经在生产环境中运行了Redis，如果重新编译和重新启动可能会给您带来问题，那么就没有必要这样做。GDB仍然可以用于对已经使用优化编译的可执行文件进行调试。

对于从Redis编译时没有进行优化导致的性能损失，您不应过于担心。这不太可能在您的环境中引起问题，因为Redis的CPU负载不高。

## 附加 GDB 到正在运行的进程

如果您已经运行了Redis服务器，您可以将GDB连接到它，这样如果Redis崩溃，就可以同时检查内部情况并生成 `core dump` 文件。

将GDB附加到Redis进程后，它将继续正常运行，不会有任何性能损失，所以这不是一个危险的操作。

为了附加GDB，您首先需要运行Redis实例的进程ID（进程的pid）。您可以使用redis-cli轻松获取它：

    $ redis-cli info | grep process_id
    process_id:58414

在上面的示例中，进程ID为**58414**。

登录到您的Redis服务器。

（可选但建议）启动`screen`或`tmux`或任何其他程序，以确保如果ssh连接超时，您的GDB会话不会被关闭。您可以在[这篇文章](http://www.linuxjournal.com/article/6340)中了解更多关于`screen`的信息。

使用以下命令将GDB附加到正在运行的Redis服务器：

    $ gdb <path-to-redis-executable> <pid>

例如：

    $ gdb /usr/local/bin/redis-server 58414

GDB将启动并附加到运行中的服务器，打印类似以下内容：

    Reading symbols for shared libraries + done
    0x00007fff8d4797e6 in epoll_wait ()
    (gdb)

在这一点上，GDB已附加，但**您的Redis实例已被GDB阻塞**。为了让Redis实例继续执行，只需在GDB提示符处键入**continue**，然后按回车键。

    (gdb) continue
    Continuing.

完成！现在您的Redis实例已经附加了GDB。现在您可以等待下一次崩溃。 :)

现在是分离你的 screen/tmux 会话的时候了，如果你是在其中运行 GDB，按下 Ctrl-a a 键。

## 现场发生事故之后

Redis有一个命令来模拟段错误（也就是一个严重错误）使用`DEBUG SEGFAULT`命令（在真正的生产实例中不要使用它！）所以我将使用这个命令来崩溃我的实例，以展示在GDB一侧发生了什么：

    (gdb) continue
    Continuing.

    Program received signal EXC_BAD_ACCESS, Could not access memory.
    Reason: KERN_INVALID_ADDRESS at address: 0xffffffffffffffff
    debugCommand (c=0x7ffc32005000) at debug.c:220
    220         *((char*)-1) = 'x';

正如你所看到的，GDB检测到Redis崩溃，并且甚至能够显示导致崩溃的文件名和行号。这比Redis崩溃报告的回溯（仅包含函数名称和二进制偏移量）要好得多。

## 获取堆栈跟踪

第一件要做的事情是用GDB获取完整的堆栈跟踪。这就像使用**bt**命令一样简单：

    (gdb) bt
    #0  debugCommand (c=0x7ffc32005000) at debug.c:220
    #1  0x000000010d246d63 in call (c=0x7ffc32005000) at redis.c:1163
    #2  0x000000010d247290 in processCommand (c=0x7ffc32005000) at redis.c:1305
    #3  0x000000010d251660 in processInputBuffer (c=0x7ffc32005000) at networking.c:959
    #4  0x000000010d251872 in readQueryFromClient (el=0x0, fd=5, privdata=0x7fff76f1c0b0, mask=220924512) at networking.c:1021
    #5  0x000000010d243523 in aeProcessEvents (eventLoop=0x7fff6ce408d0, flags=220829559) at ae.c:352
    #6  0x000000010d24373b in aeMain (eventLoop=0x10d429ef0) at ae.c:397
    #7  0x000000010d2494ff in main (argc=1, argv=0x10d2b2900) at redis.c:2046

这显示了回溯信息，但我们还想使用 **info registers** 命令转储处理器寄存器：

    (gdb) info registers
    rax            0x0  0
    rbx            0x7ffc32005000   140721147367424
    rcx            0x10d2b0a60  4515891808
    rdx            0x7fff76f1c0b0   140735188943024
    rsi            0x10d299777  4515796855
    rdi            0x0  0
    rbp            0x7fff6ce40730   0x7fff6ce40730
    rsp            0x7fff6ce40650   0x7fff6ce40650
    r8             0x4f26b3f7   1327936503
    r9             0x7fff6ce40718   140735020271384
    r10            0x81 129
    r11            0x10d430398  4517462936
    r12            0x4b7c04f8babc0  1327936503000000
    r13            0x10d3350a0  4516434080
    r14            0x10d42d9f0  4517452272
    r15            0x10d430398  4517462936
    rip            0x10d26cfd4  0x10d26cfd4 <debugCommand+68>
    eflags         0x10246  66118
    cs             0x2b 43
    ss             0x0  0
    ds             0x0  0
    es             0x0  0
    fs             0x0  0
    gs             0x0  0

请**务必将以下两个结果**都包含在您的错误报告中。

## 获取核心文件

下一步是生成核心转储，即运行中 Redis 进程的内存映像。可以使用 `gcore` 命令来完成此操作：

    (gdb) gcore
    Saved corefile core.58414

现在，你拥有核心转储文件可以发送给Redis开发者，但是**重要的是要了解**这个文件实际上包含了在Redis实例崩溃时所有的数据；Redis开发者将确保不与其他人分享此内容，并在不再用于调试目的时立即删除该文件，但请注意，通过发送核心文件，您正在发送您的数据。

## 给开发者发送什么

最后，你可以把所有东西发送给Redis核心团队：

**Redis** 可执行文件的版本。
**bt** 命令产生的堆栈跟踪和寄存器转储。
通过 gdb 生成的核心文件。
操作系统和 GCC 版本的信息，以及您正在使用的 Redis 版本。

## 谢谢

你的帮助非常重要！很多问题只能通过这种方式跟踪。所以非常感谢！
