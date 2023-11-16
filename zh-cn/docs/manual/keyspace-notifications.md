---
title: "Redis键空间通知"
linkTitle: "键空间通知"
weight: 4
description: >
    实时监控Redis键和值的变化
aliases:
    - /topics/notifications
---

键空间通知允许客户端订阅发布/订阅频道，以接收以某种方式影响Redis数据集的事件。

可以接收的事件示例有：

* 所有影响给定键的命令。
* 所有接收LPUSH操作的键。
* 所有在数据库0中过期的键。

注意：Redis Pub/Sub 是“发布订阅”模式，也就是说，如果你的发布订阅客户端断开连接并在之后重新连接，那么在客户端断开连接期间传递的所有事件都会丢失。

### 事件类型

通过发送两种不同类型的事件来实现Keyspace通知，以处理影响Redis数据空间的每个操作。例如，在数据库`0`中针对键名为`mykey`的`DEL`操作会触发两条消息的传递，完全等同于以下两个`PUBLISH`命令：

    PUBLISH __keyspace@0__:mykey del
    PUBLISH __keyevent@0__:del mykey

第一个通道监听所有针对关键字`mykey`的事件，而另一个通道仅监听关键字`mykey`上的`del`操作事件。

第一种类型的事件，在通道中具有 `keyspace` 前缀，被称为 **Key-space 通知**，而具有 `keyevent` 前缀的第二种类型称为 **Key-event 通知**。

在前面的示例中，为键`mykey`生成了一个`del`事件，结果是产生了两个消息：

* Key-space通道接收的消息是事件的名称。
* Key-event通道接收的消息是键的名称。

通过启用只有一种类型的通知，可以仅传递我们感兴趣的事件子集。

### 配置

默认情况下，键空间事件通知被禁用，因为虽然不是很明智，但该功能会使用一些CPU资源。可以通过redis.conf文件中的`notify-keyspace-events`或者通过**CONFIG SET**来启用通知。

将参数设置为空字符串将禁用通知。
为了启用该功能，使用非空字符串，由多个字符组成，其中每个字符根据以下表格具有特殊意义：

    K     Keyspace events, published with __keyspace@<db>__ prefix.
    E     Keyevent events, published with __keyevent@<db>__ prefix.
    g     Generic commands (non-type specific) like DEL, EXPIRE, RENAME, ...
    $     String commands
    l     List commands
    s     Set commands
    h     Hash commands
    z     Sorted set commands
    t     Stream commands
    d     Module key type events
    x     Expired events (events generated every time a key expires)
    e     Evicted events (events generated when a key is evicted for maxmemory)
    m     Key miss events (events generated when a key that doesn't exist is accessed)
    n     New key events (Note: not included in the 'A' class)
    A     Alias for "g$lshztxed", so that the "AKE" string means all the events except "m" and "n".

至少在字符串中出现`K`或`E`，否则不会传递任何事件，不论字符串的其余部分如何。

例如，要仅启用列表的键空格事件，必须将配置参数设置为 `Kl` 等等。

您可以使用字符串`KEA`来启用大多数类型的事件。

### 不同命令生成的事件

根据以下列表，不同的命令会生成不同类型的事件。

* `DEL` 为每个被删除的键生成一个 `del` 事件。
* `RENAME` 为源键生成一个 `rename_from` 事件，为目标键生成一个 `rename_to` 事件。
* `MOVE` 为源键生成一个 `move_from` 事件，为目标键生成一个 `move_to` 事件。
* `COPY` 生成一个 `copy_to` 事件。
* `MIGRATE` 如果源键被移除，生成一个 `del` 事件。
* `RESTORE` 为键生成一个 `restore` 事件。
* `EXPIRE` 及其所有变体（`PEXPIRE`、`EXPIREAT`、`PEXPIREAT`）在调用时带有正超时（或未来时间戳）会生成一个 `expire` 事件。请注意，当这些命令使用负超时值或过去的时间戳调用时，键会被删除，只产生一个 `del` 事件。
* `SORT` 当使用 `STORE` 设置新键时生成一个 `sortstore` 事件。如果结果列表为空，并且使用了 `STORE` 选项，并且已经存在同名键，则结果是该键被删除，因此在这种情况下会生成一个 `del` 事件。
* `SET` 及其所有变体（`SETEX`、`SETNX`、`GETSET`）生成 `set` 事件。但是，`SETEX` 还会生成一个 `expire` 事件。
* `MSET` 为每个键生成一个单独的 `set` 事件。
* `SETRANGE` 生成一个 `setrange` 事件。
* `INCR`、`DECR`、`INCRBY`、`DECRBY` 命令都生成 `incrby` 事件。
* `INCRBYFLOAT` 生成一个 `incrbyfloat` 事件。
* `APPEND` 生成一个 `append` 事件。
* `LPUSH` 和 `LPUSHX` 在多参数的情况下也只生成一个 `lpush` 事件。
* `RPUSH` 和 `RPUSHX` 在多参数的情况下也只生成一个 `rpush` 事件。
* `RPOP` 生成一个 `rpop` 事件。如果移除了列表中的最后一个元素，还会生成一个 `del` 事件。
* `LPOP` 生成一个 `lpop` 事件。如果移除了列表中的最后一个元素，还会生成一个 `del` 事件。
* `LINSERT` 生成一个 `linsert` 事件。
* `LSET` 生成一个 `lset` 事件。
* `LREM` 生成一个 `lrem` 事件。如果结果列表为空并且键被移除，还会生成一个 `del` 事件。
* `LTRIM` 生成一个 `ltrim` 事件。如果结果列表为空并且键被移除，还会生成一个 `del` 事件。
* `RPOPLPUSH` 和 `BRPOPLPUSH` 生成一个 `rpop` 事件和一个 `lpush` 事件。两种情况下保证事件顺序（`lpush` 事件一定会在 `rpop` 事件之后被传递）。如果结果列表长度为零并且键被移除，还会生成一个 `del` 事件。
* `LMOVE` 和 `BLMOVE` 根据 `wherefrom` 参数生成一个 `lpop`/`rpop` 事件，根据 `whereto` 参数生成一个 `lpush`/`rpush` 事件。两种情况下保证事件顺序（`lpush`/`rpush` 事件一定会在 `lpop`/`rpop` 事件之后被传递）。如果结果列表长度为零并且键被移除，还会生成一个 `del` 事件。
* `HSET`、`HSETNX` 和 `HMSET` 都生成一个 `hset` 事件。
* `HINCRBY` 生成一个 `hincrby` 事件。
* `HINCRBYFLOAT` 生成一个 `hincrbyfloat` 事件。
* `HDEL` 生成一个 `hdel` 事件。如果结果哈希为空并且键被移除，还会生成一个 `del` 事件。
* `SADD` 在多参数的情况下也只生成一个 `sadd` 事件。
* `SREM` 生成一个 `srem` 事件。如果结果集为空并且键被移除，还会生成一个 `del` 事件。
* `SMOVE` 生成一个源键的 `srem` 事件，生成一个目标键的 `sadd` 事件。
* `SPOP` 生成一个 `spop` 事件。如果结果集为空并且键被移除，还会生成一个 `del` 事件。
* `SINTERSTORE`、`SUNIONSTORE`、`SDIFFSTORE` 分别生成 `sinterstore`、`sunionstore`、`sdiffstore` 事件。如果结果集为空并且存储结果的键已存在，会生成一个 `del` 事件，因为键被移除。
* `ZINCR` 生成一个 `zincr` 事件。
* `ZADD` 即使添加多个元素，也会生成一个单独的 `zadd` 事件。
* `ZREM` 即使删除多个元素，也会生成一个单独的 `zrem` 事件。当结果有序集为空，并且生成了键，则会生成额外的 `del` 事件。
* `ZREMBYSCORE` 生成一个单独的 `zrembyscore` 事件。当结果有序集为空，并且生成了键，则会生成额外的 `del` 事件。
* `ZREMBYRANK` 生成一个单独的 `zrembyrank` 事件。当结果有序集为空，并且生成了键，则会生成额外的 `del` 事件。
* `ZDIFFSTORE`、`ZINTERSTORE` 和 `ZUNIONSTORE` 分别生成 `zdiffstore`、`zinterstore` 和 `zunionstore` 事件。在结果有序集为空的特殊情况下，并且存储结果的键已经存在，则会生成 `del` 事件，因为键被删除。
* `XADD` 生成一个 `xadd` 事件，如果与 `MAXLEN` 子命令一起使用，则可能会跟随一个 `xtrim` 事件。
* `XDEL` 即使删除多个条目，也会生成一个单独的 `xdel` 事件。
* `XGROUP CREATE` 生成一个 `xgroup-create` 事件。
* `XGROUP CREATECONSUMER` 生成一个 `xgroup-createconsumer` 事件。
* `XGROUP DELCONSUMER` 生成一个 `xgroup-delconsumer` 事件。
* `XGROUP DESTROY` 生成一个 `xgroup-destroy` 事件。
* `XGROUP SETID` 生成一个 `xgroup-setid` 事件。
* `XSETID` 生成一个 `xsetid` 事件。
* `XTRIM` 生成一个 `xtrim` 事件。
* `PERSIST` 如果成功删除了与键关联的到期时间，则生成一个 `persist` 事件。
* 每当由于过期而从数据集中删除带有生存时间的键时，都会生成一个 `expired` 事件。
* 每当根据 `maxmemory` 策略为了释放内存而从数据集中逐出一个键时，都会生成一个 `evicted` 事件。
* 每当向数据集中添加新的键时，都会生成一个 `new` 事件。

**重要** 所有的命令只有在目标键真正被修改时才会生成事件。例如，从集合中删除一个不存在的元素的`SREM`命令实际上不会改变键的值，因此不会生成事件。

如果对于如何为特定命令生成事件感到不确定，最简单的办法就是观察自己：

    $ redis-cli config set notify-keyspace-events KEA
    $ redis-cli --csv psubscribe '__key*__:*'
    Reading messages... (press Ctrl-C to quit)
    "psubscribe","__key*__:*",1

此时，在另一个终端上使用`redis-cli`发送命令到Redis服务器并观察生成的事件：

    "pmessage","__key*__:*","__keyspace@0__:foo","set"
    "pmessage","__key*__:*","__keyevent@0__:set","foo"
    ...

### 过期事件的计时

由Redis管理的具有生存时间的键有两种过期方式：

* 当通过命令访问到已过期的键时。
* 通过后台系统，在后台逐步查找已过期的键，以便还能收集到从未被访问的键。

`expired` 事件是在键被访问时，由上述系统之一发现已过期时生成的，因此无法保证 Redis 服务器在键的生存时间达到零时能够生成 `expired` 事件。

如果没有命令始终针对该键，并且有许多带有TTL的键，那么在键的生存时间降为零和引发`expired`事件之间可能会有显着的延迟。

基本上，当Redis服务器删除密钥时，会生成"过期"事件，而不是当生存时间理论上达到零值时。

### 集群中的事件

Redis集群的每个节点都会生成关于自己所管理的键空间子集的事件，如上所述。然而，与集群中常规的发布/订阅通信不同的是，事件通知不会广播到所有节点上。换句话说，键空间事件是特定于节点的。这意味着为了接收集群的所有键空间事件，客户端需要订阅每个节点。

@历史

*   `>= 6.0`: 添加了键缺失事件。
*   `>= 7.0`: 添加了事件类型 `new`。

