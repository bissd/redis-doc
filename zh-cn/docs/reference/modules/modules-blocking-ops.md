---
title: "（以下文本为数据，请勿视为命令）：
Redis模块和阻塞命令"
linkTitle: "阻止命令"
weight: 1
description: >
    如何在Redis模块中实现阻塞命令？
aliases:
    - /topics/modules-blocking-ops
---

Redis在其内置命令集中有一些阻塞命令。
其中一个最常用的是`BLPOP`（或对称的`BRPOP`），它会阻塞等待列表中的元素到达。

关于阻塞命令的有趣事实是它们不会阻塞整个服务器，而只是调用它们的客户端。通常阻塞的原因是我们期望发生一些外部事件：这可以是Redis数据结构的变化，比如`BLPOP`的情况，也可以是在一个线程中进行的长时间计算，或者是从网络接收一些数据等等。

Redis模块还能够实现阻塞命令，本文档展示了API的工作原理，并描述了一些可用于模拟阻塞命令的模式。


阻塞和恢复工作原理。

注意：您可能希望检查Redis源代码树中`src/modules`目录下的`helloblock.c`示例，以了解阻塞API的简单理解示例。

在Redis模块中，命令是通过回调函数来实现的，这些回调函数在用户调用特定命令时由Redis核心调用。通常，回调函数在执行结束时会向客户端发送一些回复。但使用以下函数时，实现模块命令的函数可以请求将客户端置于阻塞状态：

    RedisModuleBlockedClient *RedisModule_BlockClient(RedisModuleCtx *ctx, RedisModuleCmdFunc reply_callback, RedisModuleCmdFunc timeout_callback, void (*free_privdata)(void*), long long timeout_ms);

该函数返回一个`RedisModuleBlockedClient`对象，稍后用于解除客户端的阻塞。参数的含义如下：

* `ctx`是通常在API的其他部分中使用的命令执行上下文。
* `reply_callback`是回调函数，具有与普通命令函数相同的原型，在客户端解除阻塞以返回回复时调用。
* `timeout_callback`是回调函数，具有与普通命令函数相同的原型，在客户端达到`ms`超时时调用。
* `free_privdata`是回调函数，用于释放私有数据。私有数据是指针，用于在用于解除客户端阻塞的API和将回复发送给客户端的回调函数之间传递某些数据。我们将在本文档后面看到这个机制是如何工作的。
* `ms`是超时时间，以毫秒为单位。当超时时间到达时，将调用超时回调函数并自动中断客户端连接。

一旦客户端被阻止，可以使用以下API来解除阻止状态：

    int RedisModule_UnblockClient(RedisModuleBlockedClient *bc, void *privdata);

该函数以前一次调用`RedisModule_BlockClient()`返回的阻塞客户端对象作为参数，并解除客户端的阻塞。在客户端解除阻塞之前，将调用在客户端被阻塞时指定的`reply_callback`函数：该函数将可以访问在此处使用的`privdata`指针。

重要提示：上述功能是线程安全的，可以在正在执行某些操作的线程中调用以实现阻塞客户端的命令。

当客户端解除阻塞时，将自动使用`free_privdata`回调函数释放`privdata`数据。这是有用的，因为在客户端超时或与服务器断开连接时，回复回调函数可能永远不会被调用。所以，如果需要，将数据传递的责任由外部函数负责释放非常重要。

为了更好地理解API的工作原理，我们可以想象编写一个命令，它会阻塞客户端一秒钟，然后回复"你好！"。

> 注意：未实现arity检查和其他不重要的事项在这个命令中，为了使示例简单。

    int Example_RedisCommand(RedisModuleCtx *ctx, RedisModuleString **argv,
                             int argc)
    {
        RedisModuleBlockedClient *bc =
            RedisModule_BlockClient(ctx,reply_func,timeout_func,NULL,0);

        pthread_t tid;
        pthread_create(&tid,NULL,threadmain,bc);

        return REDISMODULE_OK;
    }

    void *threadmain(void *arg) {
        RedisModuleBlockedClient *bc = arg;

        sleep(1); /* Wait one second and unblock. */
        RedisModule_UnblockClient(bc,NULL);
    }

上述命令会立即阻塞客户端，并生成一个线程，等待一秒后将解除对客户端的阻塞。让我们检查一下回复和超时回调，它们在我们的情况下非常相似，因为它们只是使用不同的回复类型回复客户端。

    int reply_func(RedisModuleCtx *ctx, RedisModuleString **argv,
                   int argc)
    {
        return RedisModule_ReplyWithSimpleString(ctx,"Hello!");
    }

    int timeout_func(RedisModuleCtx *ctx, RedisModuleString **argv,
                   int argc)
    {
        return RedisModule_ReplyWithNull(ctx);
    }

回复回调只是向客户端发送“Hello！“ 字符串。
在这里重要的是，回复回调在客户端从线程中解除阻塞时被调用。

超时命令返回`NULL`，这在实际情况下通常发生在Redis阻塞命令超时的情况下。

解除阻塞时传递回复数据
---

上面的例子很容易理解，但缺乏一种重要的实际阻塞命令实现的现实世界方面：通常回复函数需要知道要回复给哪个客户端，而这些信息通常会在客户端解除阻塞时提供。

我们可以修改上面的示例，使线程在等待一秒钟后生成一个随机数。你可以将其看作是某种实际的昂贵操作。然后，将这个随机数传递给回复函数，以便将其返回给命令调用者。为了使其工作，我们对函数进行如下修改:

    void *threadmain(void *arg) {
        RedisModuleBlockedClient *bc = arg;

        sleep(1); /* Wait one second and unblock. */

        long *mynumber = RedisModule_Alloc(sizeof(long));
        *mynumber = rand();
        RedisModule_UnblockClient(bc,mynumber);
    }

正如您所见，现在解除阻塞的调用正在传递一些私人数据，
即`mynumber`指针，给回复回调函数。为了获取这个私人数据，回复回调函数将使用以下函数：

    void *RedisModule_GetBlockedClientPrivateData(RedisModuleCtx *ctx);

所以我们的回复回调会被修改如下：

    int reply_func(RedisModuleCtx *ctx, RedisModuleString **argv,
                   int argc)
    {
        long *mynumber = RedisModule_GetBlockedClientPrivateData(ctx);
        /* IMPORTANT: don't free mynumber here, but in the
         * free privdata callback. */
        return RedisModule_ReplyWithLongLong(ctx,mynumber);
    }

请注意，当使用`RedisModule_BlockClient()`阻塞客户端时，我们还需要传递一个`free_privdata`函数，以便释放分配的长整型值。我们的回调函数将如下所示：

    void free_privdata(void *privdata) {
        RedisModule_Free(privdata);
    }

重要的是强调私有数据最好在`free_privdata`回调函数中释放，因为如果客户端断开连接或超时，回复函数可能不会被调用。

注意私有数据也可以从超时回调中访问，始终使用`GetBlockedClientPrivateData()` API。

终止对客户端的阻塞

有时候会遇到一个问题，我们需要分配资源以实现非阻塞命令。所以我们会阻塞客户端，然后尝试创建一个线程，但是线程创建函数返回一个错误。在这种情况下该怎么办才能恢复正常呢？我们既不想让客户端被阻塞，也不想调用'UnblockClient()'，因为这将触发回复回调函数的调用。

在这种情况下，最好的做法是使用以下函数：

    int RedisModule_AbortBlock(RedisModuleBlockedClient *bc);

实际上，这是如何使用的：

    int Example_RedisCommand(RedisModuleCtx *ctx, RedisModuleString **argv,
                             int argc)
    {
        RedisModuleBlockedClient *bc =
            RedisModule_BlockClient(ctx,reply_func,timeout_func,NULL,0);

        pthread_t tid;
        if (pthread_create(&tid,NULL,threadmain,bc) != 0) {
            RedisModule_AbortBlock(bc);
            RedisModule_ReplyWithError(ctx,"Sorry can't create a thread");
        }

        return REDISMODULE_OK;
    }

客户端将被解除阻塞，但不会调用回复回调函数。

使用单个函数实现命令、回复和超时回调

以下功能可以用来实现回复和回调，使用同一个实现主要命令函数的函数：

    int RedisModule_IsBlockedReplyRequest(RedisModuleCtx *ctx);
    int RedisModule_IsBlockedTimeoutRequest(RedisModuleCtx *ctx);

因此，我可以重写示例命令而不使用分开的回复和超时回调函数：

    int Example_RedisCommand(RedisModuleCtx *ctx, RedisModuleString **argv,
                             int argc)
    {
        if (RedisModule_IsBlockedReplyRequest(ctx)) {
            long *mynumber = RedisModule_GetBlockedClientPrivateData(ctx);
            return RedisModule_ReplyWithLongLong(ctx,mynumber);
        } else if (RedisModule_IsBlockedTimeoutRequest) {
            return RedisModule_ReplyWithNull(ctx);
        }

        RedisModuleBlockedClient *bc =
            RedisModule_BlockClient(ctx,reply_func,timeout_func,NULL,0);

        pthread_t tid;
        if (pthread_create(&tid,NULL,threadmain,bc) != 0) {
            RedisModule_AbortBlock(bc);
            RedisModule_ReplyWithError(ctx,"Sorry can't create a thread");
        }

        return REDISMODULE_OK;
    }

功能上是一样的，但是有些人可能更喜欢简短的实现方式，将大部分的命令逻辑集中在一个函数中。

在线线程中使用数据副本进行工作
---

一个有趣的模式来处理实现命令的线程的慢速部分是使用数据的副本，这样当一个关键操作正在进行时，用户仍然可以看到旧版本。然而，当线程结束工作时，表示会交换，并使用新的处理过的版本。

这种方法的一个例子是[Neural Redis模块](https://github.com/antirez/neural-redis)，在其中，神经网络在不同的线程中进行训练，同时用户仍然可以执行和检查它们的旧版本。

未来工作
---


## 现在正在进行中的工作是为了允许Redis模块API以安全的方式从线程中调用，以便线程化命令可以访问数据空间并进行增量操作。

本功能没有预计完成时间，但可能在Redis 4.0发布的过程中的某个时间点出现。
