---
title: 事务
linkTitle: 事务
weight: 30
description: Redis中的事务工作原理
aliases:
  - /topics/transactions
  - /docs/manual/transactions/
---

Redis的事务允许将一组命令在单步中执行，这些命令主要围绕`MULTI`, `EXEC`, `DISCARD` 和 `WATCH`。
Redis的事务提供两个重要的保证：

* 事务中的所有命令都被序列化并按顺序执行。另一个客户端发送的请求永远不会在Redis Transaction执行**中间**被服务。
  这保证了命令被执行为单个隔离操作。

* `EXEC` 命令触发事务中所有命令的执行，因此如果客户端在调用`EXEC` 命令之前在事务上下文中丢失到服务器的连接，则不会执行任何操作，
  相反，如果called `EXEC` command ，所有命令都会执行。在使用[仅追加文件](/topics/persistence#append-only-file)时，Redis确保
  使用单个write(2)系统调用将事务写入磁盘。然而，如果Redis服务器崩溃或由系统管理员以某种硬方式杀死，可能只会注册部分操作。
  Redis会在重启后检测到这种条件，并会退出并报错。使用`redis-check-aof`工具，可以修复仅追加文件，
  这将删除部分事务，使服务器可以再次启动。

从2.2版本开始，Redis允许为上述两者提供额外的保证，以类似于check-and-set（CAS）操作的方式提供乐观锁定。
这在该页面的[后面](#cas)有文档记录。

## 使用

使用`MULTI`命令进入Redis事务。此命令始终会回复`OK`。此时用户可以发布多个命令。Redis将对这些命令进行排队，而不是执行它们。
`EXEC` 被调用时执行所有的命令。

调用`DISCARD` 将清空事务队列并退出事务。

以下示例原子性地递增键`foo` 和`bar`。

```
> MULTI
OK
> INCR foo
QUEUED
> INCR bar
QUEUED
> EXEC
1) (integer) 1
2) (integer) 1
```

如上面的会话中所明显看到的，`EXEC`返回一个回复数组，其中每个元素是事务中单个命令的回复，按照发出命令的顺序。

当Redis连接处于`MULTI`请求的上下文中时，所有命令都会回复字符串`QUEUED`（从Redis协议的角度来看，这是作为状态回复发送的）。
在调用`EXEC`时，排队的命令只是被安排执行。

## 事务中的错误

在事务过程中，可能会遇到两种类型的命令错误：

* 命令可能无法排队，因此在调用`EXEC` 之前可能会出现错误。 举例来说，命令可能在语法上错误（参数个数错误，命令名错误，...），或者可能存在一些严重的情况，比如内存不足（如果服务器配置有内存限制，使用`maxmemory`指令）。
* 在调用`EXEC`后，命令可能失败，例如，我们针对错误的值进行了操作（例如，对字符串值调用列表操作）。

从Redis 2.6.5版本开始，服务器将在积累命令期间检测错误。 然后，服务器会拒绝执行事务，并在`EXEC`期间返回错误，丢弃事务。

>**对Redis < 2.6.5的注意事项：** 在Redis 2.6.5之前，客户端需要通过检查排队命令的返回值来检测在`EXEC`之前发生的错误：如果命令以QUEUED回复，则已正确排队，否则Redis返回错误。
如果在排队命令时发生错误，大多数客户机
将中止并丢弃事务。否则，如果客户端决定继续事务
`EXEC`命令将执行所有成功排队的命令，而不考虑先前的错误。

在`EXEC` *之后*发生的错误没有特殊的处理方式：所有其他命令将在事务过程中即使有些命令失败也会被执行。

这在协议级别上更清晰。在下面的示例中，即使语法正确，一个命令执行时也会失败：

```
Trying 127.0.0.1...
Connected to localhost.
Escape character is '^]'.
MULTI
+OK
SET a abc
+QUEUED
LPOP a
+QUEUED
EXEC
*2
+OK
-WRONGTYPE Operation against a key holding the wrong kind of value
```

`EXEC`返回了两个元素的的[批量字符串回复](/topics/protocol#bulk-string-reply)，其中一个是`OK`代码，另一个是错误回复。由客户端库找到对用户有意义的方法提供错误。

需要重要的是，**即使命令失败，队列中的所有其他命令也被处理** - Redis不会停止处理命令。

另一个示例，再次使用`telnet`的线协议，显示如何立即报告语法错误：

```
MULTI
+OK
INCR a b c
-ERR wrong number of arguments for 'incr' command
```

这次由于语法错误，错误的`INCR`命令根本没有排队。

## 关于回滚？

Redis不支持事务的回滚，因为支持回滚会对Redis的简单性和性能产生重大影响。

## 丢弃命令队列

可以使用`DISCARD` 来中止事务。在这种情况下，不会执行任何命令并且恢复到正常的状态。

```
> SET foo 1
OK
> MULTI
OK
> INCR foo
QUEUED
> DISCARD
OK
> GET foo
"1"
```
<a name="cas"></a>

## 使用check-and-set实现乐观锁定

`WATCH`用于为Redis事务提供check-and-set（CAS）行为。

监控`WATCH`的键以检测有针对性的变化。如果在`EXEC`命令前至少有一个监控的键被修改，整个事务将中止，并且`EXEC`返回[空回复](/topics/protocol#nil-reply)以通知事务失败。

例如，假设我们需要原子性地将键的值递增1（假设Redis没有`INCR`）。

首次尝试可能如下：

```
val = GET mykey
val = val + 1
SET mykey $val
```

只有当我们在给定时间内有一个单独的客户端执行操作时，这才能可靠地工作。如果多个客户端尝试在大约相同的时间内递增键，将会出现竞态条件。例如，
客户端A和B将读取旧值，例如10。这个值将被两个客户端增加到11，最后设置为键的值。因此，最终的值将是11而不是12。

多亏了`WATCH`，我们能够非常好地模拟这个问题：

```
WATCH mykey
val = GET mykey
val = val + 1
MULTI
SET mykey $val
EXEC
```

使用上述代码，如果存在竞态条件，并且另一个客户端修改了“val”的结果，在我们调用`WATCH`和我们调用`EXEC`之间的时间，交易将失败。

我们只需要重复操作，希望这次我们不会得到新的竞态。这种形式的锁定被称为_乐观锁定_。
在许多用例中，多个客户端将访问不同的键，因此冲突是不太可能的 - 通常不需要重复操作。

## WATCH 解释

那么什么是`WATCH` 真正关于什么？它是一个让`EXEC` 具有条件性的命令：我们要求Redis仅在没有修改过`WATCH` 键的情况下执行事务。这包括
客户端的修改，比如写命令，以及Redis本身的修改，比如过期或者驱逐。如果键在被`WATCH` 和收到`EXEC`之间被修改，
整个事务将被中止替代。

**注**
* 在Redis 6.0.9之前的版本中，过期键不会导致事务被中止。 [关于这个的更多信息](https://github.com/redis/redis/pull/7920)
* 事务内的命令不会触发`WATCH`条件，因为它们只是在发送`EXEC` 时排队。

`WATCH`可以被多次调用。简单地说，所有的`WATCH`调用效果都是从调用开始，一直到`EXEC` 被调用的时候监视变化。你也可以发送任意数量的键到
单个的`WATCH` 调用。

当`EXEC`被调用时，所有键都会被`UNWATCH`，无论事务是否被中止。 并且，当客户端连接关闭时，一切都会被`UNWATCH`。

也可以使用`UNWATCH`命令（没有参数）来清除所有监视过的键。有时这很有用，因为我们对几个键进行了乐观锁定，可能需要执行事务来改变那些键，但是在读取键的当前内容后，我们不希望继续进行。 当这种情况发生时，我们只需调用
`UNWATCH`，从而可以将连接直接自由地用于新的事务。

### 使用WATCH 实现ZPOP

为了说明如何使用`WATCH`创建新的原子操作，否则Redis不支持，一种好办法是实现ZPOP
（`ZPOPMIN`, `ZPOPMAX` 及其阻塞变体只在5.0版本中添加），即以原子方式从排序集中弹出低分数的元素。这是最简单的
实现：

```
WATCH zset
element = ZRANGE zset 0 0
MULTI
ZREM zset element
EXEC
```

如果`EXEC` 失败（即返回[空回复](/topics/protocol#nil-reply)），我们只需重复操作。

## Redis 脚本和事务

在考虑在redis中进行类似事务的操作时，另外需要考虑的是[redis 脚本](/commands/eval)，它们是事务性的。你能在Redis事务中做的所有事情，
你也可以在脚本中做，通常，脚本会更简单和更快。
