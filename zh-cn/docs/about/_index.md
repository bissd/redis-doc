---
title: Redis简介
linkTitle: "关于"
weight: 10
description: 了解Redis开源项目
aliases:
  - /topics/introduction
  - /buzz
---

Redis 是一个开源 (BSD 许可证)、内存中的数据结构存储，用作数据库、缓存、消息代理和流处理引擎。Redis 提供了字符串、哈希、列表、集合、有序集合、位图、HyperLogLogs、地理空间索引和流等[数据结构](/docs/data-types/)。Redis 内置了[复制](/topics/replication)、[Lua 脚本](/commands/eval)、[LRU 淘汰](/docs/reference/eviction/)、[事务](/topics/transactions)、不同级别的[持久化](/topics/persistence)，并通过[Redis Sentinel](/topics/sentinel)提供高可用性，并通过[Redis Cluster](/topics/cluster-tutorial)进行自动分区。

你可以在这些类型上运行 __原子操作__ ，比如[向字符串追加](/commands/append)；
[递增哈希中的值](/commands/hincrby)；[向列表中推送元素](/commands/lpush)；
[计算集合的交集](/commands/sinter)、[并集](/commands/sunion)和[差集](/commands/sdiff)；
或者[获取排序集中排名最高的成员](/commands/zrange)。

要实现最佳性能，Redis与**内存数据集**配合工作。根据您的用例，Redis可以通过定期[将数据集转储到磁盘](/topics/persistence#snapshotting)或通过[将每个命令附加到基于磁盘的日志](/topics/persistence#append-only-file)来持久化数据。如果您只需要一个功能丰富的网络化内存缓存，还可以禁用持久化。

Redis支持[异步复制](/topics/replication)，具有快速的非阻塞同步和在网络分裂时的自动重新连接和部分重新同步功能。

Redis 还包含有：

* [事务](/topics/transactions)
* [发布与订阅](/topics/pubsub)
* [Lua 脚本](/commands/eval)
* [有限生存时间的键](/commands/expire)
* [LRU 键驱逐](/docs/reference/eviction)
* [自动故障转移](/topics/sentinel)

您可以从[大多数编程语言](/clients)使用Redis。

Redis 是用 **ANSI C** 写的，并且可以在大多数 POSIX 系统上工作，如 Linux、\*BSD 和 Mac OS X，而无需外部依赖。Linux 和 OS X 是 Redis 开发和测试最多的两个操作系统，并且我们**推荐在部署中使用 Linux**。Redis 可能在类似 SmartOS 的基于 Solaris 系统上工作，但支持是*尽力而为*的。没有针对 Windows 构建的官方支持。

<hr>
