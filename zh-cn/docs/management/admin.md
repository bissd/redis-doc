---
title: Redis 管理
linkTitle: 管理
weight: 1
description: 关于在生产环境中配置和管理Redis的建议
aliases: [
    /topics/admin,
    /topics/admin.md,
    /manual/admin,
    /manual/admin.md
]
---

## Redis设置技巧

### Linux

* 使用Linux操作系统部署Redis。Redis还在OS X上进行了测试，并且不时在FreeBSD和OpenBSD系统上进行测试。但是，Linux是大多数压力测试和生产部署的主要平台。

*将Linux内核过度承诺内存设置为1。在`/etc/sysctl.conf`中添加`vm.overcommit_memory = 1`。 然后，重新启动或运行命令`sysctl vm.overcommit_memory=1`以激活设置。 详细信息请参见[FAQ: 在Linux上出现fork()错误的背景保存失败？](https://redis.io/docs/getting-started/faq/#background-saving-fails-with-a-fork-error-on-linux)。

在确保Linux内核特性Transparent Huge Pages不影响Redis内存使用和延迟的情况下运行以下命令以禁用它：`echo never > /sys/kernel/mm/transparent_hugepage/enabled`。有关更多上下文信息，请参阅[延迟诊断 - 由透明巨页面引起的延迟](https://redis.io/docs/management/optimization/latency/#latency-induced-by-transparent-huge-pages)。

### 记忆

* 确保交换空间已启用，并且您的交换文件大小等于系统上的内存量。如果 Linux 没有设置交换空间，并且您的 Redis 实例意外消耗了过多的内存，Redis 可能在内存用尽时崩溃，或者 Linux 内核的 OOM 杀手会杀死 Redis 进程。当交换空间被启用时，您可以检测到延迟峰值并采取相应措施。

* 在您的实例中设置明确的`maxmemory`选项限制，以确保在系统内存接近达到限制时，它会报告错误而不是失败。请注意，`maxmemory`应通过计算Redis的开销（而不仅仅是数据）和碎片开销来设置。因此，如果您认为有10 GB的空闲内存，将其设置为8或9。

* 如果您在一个写入较多的应用程序中使用Redis，在将RDB文件保存到磁盘或重新编写AOF日志时，Redis的内存使用量可能是正常情况下的两倍。使用的额外内存与保存过程中写入操作修改的内存页面的数量成比例，因此通常与在此期间访问的键的数量（或聚合类型的项）成比例。请确保适当调整内存大小。

* 查看 `LATENCY DOCTOR` 和 `MEMORY DOCTOR` 命令以协助故障排除。

### 图像处理

* 在使用daemontools时，使用`daemonize no`。

### 复制

* 根据 Redis 使用的内存量设置一个比较复杂的复制积压，积压可以使复制实例更容易与主（Master）实例进行同步。

* 如果使用复制功能，即使持久化被禁用，Redis也会执行RDB保存。（这不适用于无盘复制。）如果主节点上没有磁盘使用，请启用无盘复制。

* 如果您正在使用复制功能，请确保主服务器已启用持久化，或者在崩溃后不会自动重启。副本将尝试保持与主服务器的完全一致，因此如果主服务器以空数据集重新启动，副本也将被清除。

### 安全

* 默认情况下，Redis不需要任何身份验证，且监听所有网络接口。如果您在互联网或其他攻击者可以到达的地方暴露Redis，这将是一个很大的安全问题。请参阅[此次攻击](http://antirez.com/news/96)以了解它可能有多危险。请查看我们的[安全页面](/topics/security)和[快速入门](/topics/quickstart)了解有关如何保护Redis的信息。

# 在EC2上运行Redis

* 使用基于HVM的实例，而不是基于PV的实例。
* 不要使用旧的实例系列。例如，使用基于HVM的m3.medium替代基于PV的m1.medium。
* 注意处理在使用EC2 EBS卷时与Redis持久性相关的问题，因为有时EBS卷具有较高的延迟特性。
* 如果在副本与主服务器同步时出现问题，您可能希望尝试新的无盘复制功能。

## 升级或重新启动 Redis 实例而无需停机

Redis被设计为在服务器上长时间运行的进程。可以使用`CONFIG SET`命令修改许多配置选项而无需重新启动。还可以在不重新启动Redis的情况下从AOF切换到RDB快照持久性，或者反之亦然。请使用`CONFIG GET *`命令查看更多信息的输出。

不时需要重新启动，例如，升级Redis进程到更高版本时，或者需要修改一个目前不支持`CONFIG`命令的配置参数时。

按照以下步骤来避免停机时间。

* 将您的新Redis实例设置为当前Redis实例的复制品。为了实现这一点，您需要一个不同的服务器，或者一台具有足够RAM来同时运行两个Redis实例的服务器。

* 如果您使用单个服务器，请确保副本启动的端口与主实例不同，否则副本将无法启动。

* 等待复制的初始同步完成。检查副本的日志文件。

* 使用`INFO`命令，确保主节点和复制节点拥有相同数量的键。使用`redis-cli`命令检查复制节点是否按预期工作，并对您的命令进行回复。

* 允许使用`CONFIG SET slave-read-only no`向副本写入。

* 配置所有客户端使用新实例（副本）。请注意，您可能希望使用“CLIENT PAUSE”命令来确保在切换过程中没有客户端可以向旧主机写入。

* 在确认主节点不再接收任何查询之后（您可以使用“MONITOR”命令来进行检查），使用“REPLICAOF NO ONE”命令选举复制品为主节点，然后关闭您的主节点。

如果您正在使用[Redis Sentinel](/topics/sentinel)或[Redis Cluster](/topics/cluster-tutorial)，升级到较新版本的最简单方法是逐个升级副本。然后，您可以进行手动故障切换，将已升级的副本之一晋升为主节点，最后再晋升最后一个副本。

注意

Redis Cluster 4.0在集群总线协议级别上与Redis Cluster 3.2不兼容，因此在这种情况下，需要进行大规模重启。然而，Redis 5集群总线与Redis 4向后兼容。

---

