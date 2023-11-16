---
title: "事件库"
linkTitle: "事件库"
weight: 1
description: 什么是事件库？原始的Redis事件库是如何实现的？
aliases:
  - /topics/internals-eventlib
  - /topics/internals-rediseventlib
---

**注意：本文档是由Redis的创建者Salvatore Sanfilippo在Redis开发早期（约2010年）编写的，可能不反映最新的Redis实现情况。**

## 为什么需要一个事件库？

让我们通过一系列问答来弄清楚。

问题：您希望网络服务器一直在做什么？<br/>
答：监听指定端口上的传入连接，并接受它们。

Q：调用[accept](http://man.cx/accept%282%29)会得到一个描述符。我该如何处理它？<br/>
A：保存描述符并在其上进行非阻塞读/写操作。

Q: 为什么读/写操作必须是非阻塞的？
A: 如果文件操作（甚至是Unix中的套接字）是阻塞的，那么当服务器在文件I/O操作中被阻塞时，它如何接受其他连接请求呢？

Q: 我猜我需要在套接字上执行许多这样的非阻塞操作，以查看它何时准备就绪。我是对的吗？<br/>
A: 是的，这就是事件库为你做的事情。现在你明白了。

问题：事件库是如何实现其功能的？

回答：它们使用操作系统的轮询功能以及定时器。

Q: 那么是否有开源的事件库可以实现你刚才描述的功能呢？<br/>
A: 是的。`libevent`和`libev`是我记得的两个可以实现这样功能的事件库。

问题：Redis是否使用开源事件库来处理套接字I/O？

回答：不用。出于各种原因，Redis使用自己的事件库。

## Redis事件库

Redis实现了自己的事件库。事件库实现在`ae.c`中。

了解Redis事件库工作原理的最好方法是了解Redis如何使用它。

事件循环初始化
---


`initServer` 函数定义在 `redis.c` 中，它初始化了 `redisServer` 结构变量的许多字段。其中一个字段是 Redis 事件循环 `el`：

    aeEventLoop *el

`initServer` 通过调用`ae.c`中定义的`aeCreateEventLoop`来初始化`server.el`字段。`aeEventLoop`的定义如下：

    typedef struct aeEventLoop
    {
        int maxfd;
        long long timeEventNextId;
        aeFileEvent events[AE_SETSIZE]; /* Registered events */
        aeFiredEvent fired[AE_SETSIZE]; /* Fired events */
        aeTimeEvent *timeEventHead;
        int stop;
        void *apidata; /* This is used for polling API specific data */
        aeBeforeSleepProc *beforesleep;
    } aeEventLoop;

`aeCreateEventLoop`

`aeCreateEventLoop` 首先使用 `malloc` 函数分配 `aeEventLoop` 结构，然后调用 `ae_epoll.c:aeApiCreate`。

`aeApiCreate`函数`malloc`一个具有两个字段的`aeApiState`结构体，其中`epfd`保存着调用[`epoll_create`](http://man.cx/epoll_create%282%29)返回的`epoll`文件描述符，`events`是由Linux `epoll`库定义的`struct epoll_event`类型。`events`字段的使用将在后面描述。

接下来是`ae.c:aeCreateTimeEvent`。但在此之前，`initServer`调用`anet.c:anetTcpServer`创建并返回了一个_监听描述符_。该描述符默认监听*端口6379*。返回的_监听描述符_存储在`server.fd`字段中。

`aeCreateTimeEvent`
---
`aeCreateTimeEvent`

`aeCreateTimeEvent`将以下内容作为参数接受：

   * `eventLoop`：这是`redis.c`中的`server.el`。
   * milliseconds：定时器到期后与当前时间的毫秒数。
   * `proc`：函数指针。存储在定时器到期后需要调用的函数的地址。
   * `clientData`：大多数情况下为空。
   * `finalizerProc`：指向在定时事件从定时事件列表中移除之前需要调用的函数的指针。

`initServer` 调用 `aeCreateTimeEvent` 将定时事件添加到 `server.el` 的 `timeEventHead` 字段中。`timeEventHead` 是一个指向这些定时事件列表的指针。以下是 `redis.c:initServer` 函数中调用 `aeCreateTimeEvent` 的代码：

    aeCreateTimeEvent(server.el /*eventLoop*/, 1 /*milliseconds*/, serverCron /*proc*/, NULL /*clientData*/, NULL /*finalizerProc*/);

`redis.c:serverCron`执行了许多操作，帮助保持Redis的正常运行。

`aeCreateFileEvent`

[`aeCreateFileEvent`](http://man.cx/epoll_ctl) 函数的本质是执行 [`epoll_ctl`](http://man.cx/epoll_ctl) 系统调用，在由 `anetTcpServer` 创建的监听描述符上添加对 `EPOLLIN` 事件的关注，并将其与通过调用 `aeCreateEventLoop` 创建的 `epoll` 描述符关联起来。

下面是有关在`redis.c:initServer`中调用`aeCreateFileEvent`时`aeCreateFileEvent`函数的详细说明。

`initServer` 将以下参数传递给 `aeCreateFileEvent`：

  * `server.el`: 由 `aeCreateEventLoop` 创建的事件循环。`server.el` 获取了`epoll` 描述符。
  * `server.fd`: 兼作索引访问 `eventLoop->events` 表中相关文件事件结构以及存储额外信息（如回调函数）的_监听描述符_。
  * `AE_READABLE`: 表示需要监听 `server.fd` 的 `EPOLLIN` 事件。
  * `acceptHandler`: 在所监听的事件准备就绪时执行的函数。函数指针存储在 `eventLoop->events[server.fd]->rfileProc` 中。

这完成了Redis事件循环的初始化。

事件循环处理
---

`ae.c:aeMain` 在 `redis.c:main` 中被调用，它负责处理在前一阶段初始化的事件循环。

`ae.c:aeMain` 在一个循环中调用 `ae.c:aeProcessEvents`，处理待处理的时间和文件事件。

`aeProcessEvents`

`ae.c:aeProcessEvents` 通过调用 `ae.c:aeSearchNearestTimer` 在事件循环中查找最短时间内待处理的定时事件。在我们的情况下，事件循环中只有一个由 `ae.c:aeCreateTimeEvent` 创建的定时器事件。

请记住，由`aeCreateTimeEvent`创建的计时器事件可能已经过期，因为它的到期时间为一毫秒。由于计时器已经过期，`tvp` `timeval`结构变量的秒和微秒字段被初始化为零。

`tvp` 结构变量连同事件循环变量一同传递到 `ae_epoll.c:aeApiPoll`。

`aeApiPoll` 函数对 `epoll` 描述符执行 [`epoll_wait`](http://man.cx/epoll_wait)，并将详情填充到 `eventLoop->fired` 表中。

  * `fd`：现在已准备好根据掩码值进行读/写操作的描述符。
  * `mask`：现在可以对对应描述符执行的读/写事件。

`aeApiPoll` 返回准备操作的此类文件事件的数量。现在要把事情放在背景下，如果有任何客户端请求连接，`aeApiPoll` 就会注意到，并将 `eventLoop->fired` 表填充一个条目，该条目的描述符为“监听描述符”，掩码为 `AE_READABLE`。

现在，`aeProcessEvents`调用已注册为回调函数的`redis.c:acceptHandler`。`acceptHandler`在监听描述符上执行[accept](http://man.cx/accept)，并返回一个带有客户端的连接描述符。`redis.c:createClient`通过调用`ae.c:aeCreateFileEvent`在连接描述符上添加了一个文件事件，示例如下：

    if (aeCreateFileEvent(server.el, c->fd, AE_READABLE,
        readQueryFromClient, c) == AE_ERR) {
        freeClient(c);
        return NULL;
    }

`c` 是 `redisClient` 结构体变量，`c->fd` 是连接描述符。

接下来，`ae.c:aeProcessEvent` 调用了 `ae.c:processTimeEvents`。

`processTimeEvents`
---
`处理时间事件`

`ae.processTimeEvents` 迭代 `eventLoop->timeEventHead` 开始的时间事件列表。

对于每一个已经过去的定时事件，`processTimeEvents` 调用注册的回调函数。在这个例子中，它调用了唯一注册的定时事件回调函数，即 `redis.c:serverCron`。回调函数返回的是下一次需要再次调用回调函数的时间（以毫秒为单位）。这个变化通过调用`ae.c:aeAddMilliSeconds`来记录，并将在`ae.c:aeMain`的下一轮循环中进行处理。

这就是全部。
