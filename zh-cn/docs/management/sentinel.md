---
title: "高可用性与Redis Sentinel"
linkTitle: "具有Sentinel的高可用性"
weight: 4
description: 非集群Redis的高可用性
aliases: [
    /topics/sentinel,
    /docs/manual/sentinel,
    /docs/manual/sentinel.md
]
---

当不使用Redis集群时，Redis Sentinel为Redis提供高可用性。

Redis Sentinel还提供其他相关任务，例如监控、通知，并作为客户端的配置提供者。

这是Sentinel能力的完整清单，从宏观角度来看（即*大局*）：

* **监控**。Sentinel会不断检查您的主节点和副本节点是否按预期工作。
* **通知**。Sentinel可以通过API通知系统管理员或其他计算机程序，某个被监控的Redis实例出现了问题。
* **自动故障切换**。如果主节点出现问题，Sentinel可以启动故障切换过程，将一个副本提升为主节点，重新配置其他附加副本使用新的主节点，并通知使用Redis服务器的应用程序连接时使用的新地址。
* **配置提供者**。Sentinel作为客户端服务发现的权威来源：客户端连接到Sentinel以获取当前负责给定服务的Redis主节点的地址。如果发生故障切换，Sentinel将报告新的地址。

## 分布式系统中的Sentinel

Redis哨兵是一个分布式系统：

Sentinel本身被设计为在多个Sentinel进程协同工作的配置中运行。多个Sentinel进程协同工作的优势如下：

1. 当多个 Sentinel 一致认为给定的主节点不再可用时，将执行故障检测。这降低了误报的概率。
2. 即使不是所有 Sentinel 进程都在工作，Sentinel 也可以正常工作，使系统对故障具有强大的鲁棒性。毕竟，拥有自身成为单点故障的故障转移系统没有任何意义。

Sentinels、Redis实例（主节点和复制节点）以及连接到Sentinel和Redis的客户端的总和，也是具有特定属性的较大分布式系统。在本文档中，将逐步介绍概念，从理解Sentinel的基本属性所需的基本信息开始，再到理解Sentinel的工作原理所需的更复杂的信息（可选择性提供）。

## Sentinel快速开始

### 获取 Sentinel

当前版本的Sentinel被称为**Sentinel 2**。它是对最初的Sentinel实现进行的重写，使用更强大且更简单预测的算法（这些算法在本文档中有解释）。

Redis Sentinel自Redis 2.8版起提供稳定版本发布。

新的发展在“不稳定”分支中进行，并且一旦被认为是稳定的，新功能有时会被回退到最新的稳定分支中。

Redis Sentinel 1 版已经过时，并且不应该使用。这是在 Redis 2.6 中发布的。

### 运行Sentinel

如果您使用`redis-sentinel`可执行文件（或者如果您有一个名称为`redis-server`的符号链接），您可以使用以下命令行运行Sentinel：

    redis-sentinel /path/to/sentinel.conf

否则，您可以直接使用`redis-server`可执行文件，在Sentinel模式下启动它：

    redis-server /path/to/sentinel.conf --sentinel

两种方式都可以。

然而，**使用配置文件是必需的**，当运行Sentinel时，系统将使用该文件来保存当前状态，以便在重新启动时重新加载。如果没有提供配置文件或配置文件路径不可写入，Sentinel将拒绝启动。

默认情况下，Sentinel运行在**监听TCP端口26379上等待连接**，所以为了使Sentinel工作，你的服务器上的26379端口**必须打开**，以接收来自其他Sentinel实例的连接。
否则，Sentinel无法通信，无法就如何处理达成一致，因此无法执行故障转移。

### 在部署 Sentinel 之前需要了解的基本信息

1. 您需要至少三个哨兵实例来进行强大的部署。
2. 这三个哨兵实例应放置在被认为独立发生故障的计算机或虚拟机中。例如，不同的物理服务器或在不同可用区执行的虚拟机。
3. Sentinel+Redis分布式系统不能保证在故障期间保留确认的写入，因为Redis使用异步复制。然而，有一些部署Sentinel的方式使得丢失写入的窗口仅限于某些时刻，同时还有其他不太安全的部署方式。
4. 您的客户端需要支持哨兵功能。流行的客户端库支持哨兵，但不是所有的库都支持。
5. 如果不在开发环境定期进行测试，或者在生产环境更好地进行测试，没有安全的高可用设置。您可能存在配置错误，只有在太迟的时候（在夜间3点时，当您的主服务器停止工作时）才会显现出来。
6. **哨兵，Docker或其他形式的网络地址转换或端口映射应小心混合使用**：Docker执行端口重新映射，破坏了哨兵自动发现其他哨兵进程和主节点的副本列表的功能。有关更多信息，请查看本文档后面关于_哨兵和Docker_的部分。

### 配置 Sentinel

Redis源代码发布包中包含一个名为`sentinel.conf`的文件，这是一个自我说明的示例配置文件，您可以使用它来配置Sentinel。但是，典型的最小配置文件如下所示：

    sentinel monitor mymaster 127.0.0.1 6379 2
    sentinel down-after-milliseconds mymaster 60000
    sentinel failover-timeout mymaster 180000
    sentinel parallel-syncs mymaster 1

    sentinel monitor resque 192.168.1.3 6380 4
    sentinel down-after-milliseconds resque 10000
    sentinel failover-timeout resque 180000
    sentinel parallel-syncs resque 5

您只需要指定要监视的主服务器，给每个独立的主服务器（可能有任意数量的副本）一个不同的名称。不需要指定副本，它们会自动被发现。Sentinel会自动更新配置，并附加关于副本的额外信息（以便在重启时保留信息）。在故障转移期间将副本升级为主服务器，以及每次发现新的Sentinel时，配置也会被重写。

上述示例配置基本上监控了两组Redis实例，每组实例都由一个主节点和一个未定义数量的从节点组成。一组实例被称为`mymaster`，另一组实例被称为`resque`。

要理解 `sentinel monitor` 语句的参数含义，可以参考以下内容：

    sentinel monitor <master-name> <ip> <port> <quorum>

为了清楚起见，让我们逐行检查配置选项的含义：

第一行用于告诉Redis监视一个名为*mymaster*的主服务器，
该主服务器位于地址127.0.0.1和端口6379，保留数为2。除了**quorum**参数之外，一切都很明显。

* **仲裁机制（quorum）** 是指在主节点不可达时，需要多少个Sentinel达成一致，才能真正将主节点标记为故障，并在可能的情况下启动故障转移程序。
* 但是 **仲裁机制仅用于检测故障**。为了实际执行故障转移，需要从Sentinel中选择一位领导者，并授权其继续进行。这只有在 **大多数Sentinel进程的投票** 后才会发生。

所以举个例子，如果你有5个Sentinel进程，并且给定的主节点设置的仲裁值为2，那么会发生以下情况：

* 如果两个 Sentinel 在同一时间内达成一致，判断主服务器无法访问时，其中一个 Sentinel 将尝试启动故障转移。
* 如果可达的 Sentinel 至少有三个，故障转移将被授权并实际启动。

在实际应用中，这意味着在故障期间，如果大多数的 Sentinel 进程无法通信，**Sentinel 不会启动故障切换**（也就是在少数分区中不会发生故障切换）。

### 其他 Sentinel 选项

其他选项几乎总是以以下形式出现：

    sentinel <option_name> <master_name> <option_value>

* `down-after-milliseconds` 是一个实例在多少毫秒内不能被访问（要么不回复我们的 PING，要么回复错误）时，Sentinel 开始认为它已经下线。
* `parallel-syncs` 设置在故障切换后可以重新配置使用新主节点的副本数量。数字越小，故障切换过程完成的时间越长，但是如果副本配置为提供旧数据，则可能不希望所有副本同时与主节点重新同步。尽管对于副本来说，复制过程大部分是非阻塞的，但在从主节点加载批量数据时会有一个停顿的时刻。通过将此选项设置为 1，您可以确保同一时间只有一个副本无法访问。

附加选项在本文档的其余部分中进行描述，并在随 Redis 分发的示例 `sentinel.conf` 文件中记录。

运行时可以修改配置参数：

* 使用 `SENTINEL SET` 修改主服务器特定的配置参数。
* 使用 `SENTINEL CONFIG SET` 修改全局配置参数。

请参阅[_运行时重新配置哨兵_部分](#reconfiguring-sentinel-at-runtime)了解更多信息。

### 示例Sentinel部署

既然您已经了解了有关Sentinel的基本信息，您可能想知道应该将Sentinel进程放在哪里，需要多少个Sentinel进程等等。本节将展示一些示例部署。

为了以*图形化*格式向您展示配置示例，我们使用ASCII艺术，以下是不同符号的含义：

    +--------------------+
    | This is a computer |
    | or VM that fails   |
    | independently. We  |
    | call it a "box"    |
    +--------------------+

我们在方框内写上它们运行的内容：

    +-------------------+
    | Redis master M1   |
    | Redis Sentinel S1 |
    +-------------------+

不同的方框通过线条连接，以显示它们可以进行交流：

    +-------------+               +-------------+
    | Sentinel S1 |---------------| Sentinel S2 |
    +-------------+               +-------------+

网络分区显示为使用斜杠的中断线：

    +-------------+                +-------------+
    | Sentinel S1 |------ // ------| Sentinel S2 |
    +-------------+                +-------------+

还请注意：

* 主节点被称为M1、M2、M3、...、Mn。
* 从节点被称为R1、R2、R3、...、Rn（R代表replica）。
* Sentinel被称为S1、S2、S3、...、Sn。
* 客户端被称为C1、C2、C3、...、Cn。
* 当一个实例由于Sentinel的动作而改变角色时，我们用方括号括起来，所以[M1]表示一个由于Sentinel干预而成为主节点的实例。

请注意，我们永远不会显示仅使用两个 Sentinel 的设置，因为 Sentinel 始终需要与大多数进行通信以启动故障转移。

#### 示例 1：仅使用两个 Sentinel，请勿这样做

    +----+         +----+
    | M1 |---------| R1 |
    | S1 |         | S2 |
    +----+         +----+

    Configuration: quorum = 1

* 在这种设置中，如果主节点 M1 失败，由于两个 Sentinel 能够就故障达成一致（很明显设置仲裁数为1），并且也能够授权进行故障切换，所以表面上看它可能工作正常，然而请查看下一点以了解为什么这种设置是有问题的。
* 如果运行 M1 的主机停止工作，S1 也会停止工作。运行在另一个主机 S2 上的 Sentinel 将无法授权故障切换，因此系统将变得不可用。

注意，为了对不同的故障转移进行排序，需要多数认同，并随后将最新的配置传播到所有的 Sentinel 。此外，请注意，在上述设置的单侧进行故障转移而没有任何协议的能力将非常危险。

    +----+           +------+
    | M1 |----//-----| [M1] |
    | S1 |           | S2   |
    +----+           +------+

在上述配置中，我们以完全对称的方式创建了两个主节点（假设S2可以无需授权进行故障切换）。客户端可以无限期地向两个节点写入数据，一旦分区恢复，就无法确定哪个配置是正确的，以防止*永久的分区脑裂*。

所以请始终在三个不同的容器中部署至少三个哨兵。

#### 示例2：基本设置，包含三个框

这是一个非常简单的设置，优点是简单易于调整以增加安全性。它基于三个盒子，每个盒子都运行了一个Redis进程和一个Sentinel进程。


           +----+
           | M1 |
           | S1 |
           +----+
              |
    +----+    |    +----+
    | R2 |----+----| R3 |
    | S2 |         | S3 |
    +----+         +----+

    Configuration: quorum = 2

如果主服务器M1发生故障， S2和S3将会达成一致并且可以授权进行故障转移，从而使客户端能够继续使用。

在每个Sentinel设置中，由于Redis使用异步复制，总会存在一些写操作可能丢失的风险，因为可能无法将已确认的写操作传递到被提升为主服务器的副本。然而，在上述设置中，由于客户端与旧主服务器分离，会存在更高的风险，就像下面的图片所示：

             +----+
             | M1 |
             | S1 | <- C1 (writes will be lost)
             +----+
                |
                /
                /
    +------+    |    +----+
    | [M2] |----+----| R3 |
    | S2   |         | S3 |
    +------+         +----+

在这种情况下，网络分区隔离了旧主节点M1，因此副本R2被提升为主节点。然而，像C1这样与旧主节点处于同一分区的客户端可能会继续向旧主节点写入数据。由于当分区恢复时，主节点将被重新配置为新主节点的副本，丢弃其数据集，所以这些数据将永远丢失。

问题可以使用以下Redis复制功能来缓解，如果主节点检测到无法将写入传输给指定数量的副本，则可以停止接受写入。

    min-replicas-to-write 1
    min-replicas-max-lag 10

在上述配置下（请参考Redis分发中的有自注释`redis.conf`示例以获取更多信息），当Redis实例作为主节点时，如果无法向至少1个副本写入数据，则会停止接受写入操作。由于复制是异步的，*无法写入* 实际上意味着副本要么断开连接，要么在指定的`max-lag`秒内未向我们发送异步的确认消息。

使用这个配置，在上面的例子中，旧的Redis主服务器M1将在10秒后变得不可用。当分区恢复时，Sentinel配置将收敛到新配置，客户端C1将能够获取有效的配置，并继续与新主服务器进行通信。

然而没有免费的午餐。通过这种改进，如果两个复本都宕机了，主节点将停止接受写操作。这是一个权衡。

#### 示例3：客户端盒中的Sentinel

有时我们只有两个可用的Redis盒子，一个用作主节点，一个用作从节点。在这种情况下，示例2的配置是不可行的，因此我们可以采用以下方法，将Sentinels放置在客户端所在的位置：

                +----+         +----+
                | M1 |----+----| R1 |
                |    |    |    |    |
                +----+    |    +----+
                          |
             +------------+------------+
             |            |            |
             |            |            |
          +----+        +----+      +----+
          | C1 |        | C2 |      | C3 |
          | S1 |        | S2 |      | S3 |
          +----+        +----+      +----+

          Configuration: quorum = 2

在这个设置中，Sentinels的观点与客户端相同：如果大多数客户端都可以访问主节点，那就可以了。
这里的C1、C2、C3是通用的客户端，不意味着C1代表一个连接到Redis的单个客户端。更有可能是应用服务器、Rails应用程序或类似的东西。

如果M1和S1所在的机器故障，故障转移将会顺利进行，然而很容易看出，不同的网络分区将导致不同的行为。例如，如果客户端和Redis服务器之间的网络断开连接，Sentinel将无法设置，因为Redis的主服务器和副本都将无法使用。

请注意，如果C3与M1进行分区（在上述网络中几乎不可能，但在不同布局或软件层面发生故障时更有可能），我们会遇到与示例2中描述的类似问题，区别在于这里没有办法打破对称性，因为只有一个副本和主节点，所以当主节点与副本断开连接时，主节点无法停止接受查询，否则主节点在副本故障期间将永远不可用。

所以这是一个有效的设置，但是在示例2中的设置有其优势，比如Redis的HA系统在相同的盒子上运行，这可能更容易管理，并且能够对处于少数派分区中的主节点接收写入的时间进行限制。

#### 示例 4：Sentinel 客户端至少有两个以下的情况

如果客户端（例如三个网络服务器）少于三个盒子，则无法使用示例3中描述的设置。在这种情况下，我们需要采用以下混合设置。

                +----+         +----+
                | M1 |----+----| R1 |
                | S1 |    |    | S2 |
                +----+    |    +----+
                          |
                   +------+-----+
                   |            |
                   |            |
                +----+        +----+
                | C1 |        | C2 |
                | S3 |        | S4 |
                +----+        +----+

          Configuration: quorum = 3

这与示例3中的设置类似，但在这里我们在四个可用的框中运行四个Sentinels。如果主节点M1不可用，其他三个Sentinels将执行故障切换。

从理论上讲，这个设置是有效的，它会移除运行着C2和S4的盒子，并将法定人数设置为2。然而，如果我们应用层没有高可用性，我们不太可能希望在Redis端实现高可用性。

# Sentinel，Docker，NAT和可能的问题

Docker 使用一种叫做端口映射的技术：运行在 Docker 容器内的程序可能会使用不同的端口来暴露，与程序认为自己在使用的端口不同。这在同一台服务器上同时运行多个容器，并使用相同的端口非常有用。

Docker不是唯一一个发生这种情况的软件系统，还有其他网络地址转换设置，端口可能会被重新映射，有时不仅仅是端口，还有IP地址。

重新映射端口和地址会导致Sentinel存在两个问题：

1. 由于基于*hello*消息的使能自动发现其他Sentinel的方式不再有效，所以Sentinel没有办法了解地址或端口是否被重新映射，因此它会宣布错误的信息以供其他Sentinel连接。
2. 在Redis主节点的`INFO`输出中，副本的地址和端口也是以类似的方式列出的：地址通过主节点检查TCP连接的远程对等点进行检测，而端口则由副本自身在握手期间广告，但由于与第1点中所述相同的原因，端口可能是错误的。

由于 Sentinels 使用主 `INFO` 输出信息来自动检测副本，
被检测到的副本将无法连接，Sentinel 将无法对主服务器进行故障转移，
因为从系统的角度来看，没有良好的可用副本，
所以目前没有办法使用 Sentinel 监控使用 Docker 部署的一组主副本实例，
**除非你让 Docker 映射端口 1:1**。

针对第一个问题，如果您想使用Docker运行一组具有转发端口的Sentinel实例（或任何其他端口重新映射的NAT设置），您可以使用以下两个Sentinel配置指令，以便强制Sentinel宣布一组特定的IP和端口：

    sentinel announce-ip <ip>
    sentinel announce-port <port>

请注意，Docker具备在*主机网络模式*下运行的能力（有关更多信息，请查看`--net=host`选项）。由于在此设置中不重新映射端口，因此不会产生任何问题。

### IP 地址与 DNS 名称

Sentinel的旧版本不支持主机名，并要求在任何地方都要指定IP地址。
从6.2版本开始，Sentinel*可选*支持主机名。

**此功能默认已禁用。如果要启用DNS/主机名支持，请注意：**

# 1. 在您的 Redis 和 Sentinel 节点上，名称解析配置必须可靠并能够快速解析地址。地址解析的意外延迟可能对 Sentinel 产生负面影响。
# 2. 您应该在所有的 Redis 和 Sentinel 实例中都使用主机名，避免混合使用主机名和 IP 地址。为此，请分别使用 `replica-announce-ip <主机名>` 和 `sentinel announce-ip <主机名>`。

启用“resolve-hostnames”全局配置允许Sentinel接受主机名：

* 作为`sentinel monitor`命令的一部分
* 作为副本地址，若副本使用`replica-announce-ip`的主机名值。

Sentinel将接受主机名作为有效的输入并对其进行解析，但在公告实例、更新配置文件等时仍将使用IP地址作为参考。

启用`announce-hostnames`全局配置使Sentinel使用主机名。这会对客户端的回复、配置文件中的值、发给副本的`REPLICAOF`命令等产生影响。

该行为可能与某些Sentinel客户端不兼容，这些客户端可能明确要求一个IP地址。

使用主机名可能在客户端使用TLS连接实例，并需要名称而不是IP地址来执行证书ASN匹配时非常有用。

## 快速教程

在本文档的后续部分中，将逐步介绍关于[_Sentinel API_](#sentinel-api)、配置和语义的所有细节。然而，对于希望尽快使用该系统的用户，本节是一个教程，展示了如何配置和与3个 Sentinel 实例进行交互。

下面我们假设实例正在端口5000、5001和5002上执行。
我们还假设您在端口6379上运行了一个正在运行的Redis主节点，而在端口6380上运行了一个复制节点。
在本教程中，我们将在整个过程中使用IPv4回环地址127.0.0.1，假设您正在个人电脑上运行模拟。

以下是三个Sentinel配置文件的示例：

    port 5000
    sentinel monitor mymaster 127.0.0.1 6379 2
    sentinel down-after-milliseconds mymaster 5000
    sentinel failover-timeout mymaster 60000
    sentinel parallel-syncs mymaster 1

另外两个配置文件将使用5001和5002作为端口号，但内容完全相同。

关于上述配置的几点注意事项：

* `mymaster` 是主集群的名称，它用于标识主节点和它的副本。每个主集群都有不同的名称，因此 Sentinel 可以同时监控不同的主节点和副本集群。
* `quorum` 的值被设置为 2（`sentinel monitor` 配置指令的最后一个参数）。
* `down-after-milliseconds` 的值为 5000 毫秒，即 5 秒，如果我们在这段时间内未收到心跳回复，主节点将被标记为故障。


一旦您启动三个Sentinels，您将看到它们记录的一些消息，如下所示：

    +monitor master mymaster 127.0.0.1 6379 quorum 2

这是一个哨兵事件，如果您在[Pub/Sub消息](#pubsub-messages)部分中指定的事件名称上进行`SUBSCRIBE`，则可以通过Pub/Sub接收到这类事件。

Sentinel 在故障检测和故障切换过程中生成和记录不同的事件。

询问 Sentinel 关于主服务器的状态
---

使用Sentinel开始的最明显事情，是检查它监控的主节点是否正常运行：

    $ redis-cli -p 5000
    127.0.0.1:5000> sentinel master mymaster
     1) "name"
     2) "mymaster"
     3) "ip"
     4) "127.0.0.1"
     5) "port"
     6) "6379"
     7) "runid"
     8) "953ae6a589449c13ddefaee3538d356d287f509b"
     9) "flags"
    10) "master"
    11) "link-pending-commands"
    12) "0"
    13) "link-refcount"
    14) "1"
    15) "last-ping-sent"
    16) "0"
    17) "last-ok-ping-reply"
    18) "735"
    19) "last-ping-reply"
    20) "735"
    21) "down-after-milliseconds"
    22) "5000"
    23) "info-refresh"
    24) "126"
    25) "role-reported"
    26) "master"
    27) "role-reported-time"
    28) "532439"
    29) "config-epoch"
    30) "1"
    31) "num-slaves"
    32) "1"
    33) "num-other-sentinels"
    34) "2"
    35) "quorum"
    36) "2"
    37) "failover-timeout"
    38) "60000"
    39) "parallel-syncs"
    40) "1"

如您所见，它打印了有关主服务器的一些信息。其中有几个对我们来说特别重要:

1. `num-other-sentinels` 为2，因此我们知道 Sentinel 已经检测到了两个更多的 Sentinel 用于这个主节点。如果您查看日志，将会看到生成的 `+sentinel` 事件。
2. `flags` 只是 `master`。如果主节点故障，我们可以在此处预期看到 `s_down` 或 `o_down` 标志。
3. `num-slaves` 正确设置为1，则 Sentinel 也检测到我们的主节点有一个附属的副本。

为了更深入了解此实例，您可以尝试以下两个命令：

    SENTINEL replicas mymaster
    SENTINEL sentinels mymaster

第一个将提供与主节点连接的副本相关的类似信息，第二个则涉及其他 Sentinel。

获取当前主节点的地址
---

如我们已经指定的，Sentinel还充当了一个配置提供者，供希望连接到一组主服务器和副本的客户端使用。由于可能的故障切换或重新配置，客户端对于一组实例中当前活动的主服务器没有任何概念，因此Sentinel提供了一个API来提问这个问题：

    127.0.0.1:5000> SENTINEL get-master-addr-by-name mymaster
    1) "127.0.0.1"
    2) "6379"

### 测试故障切换

此时，我们的玩具哨兵部署已经准备好进行测试了。我们可以直接杀掉我们的主节点，并检查配置是否发生更改。为此，我们只需执行以下操作：

    redis-cli -p 6379 DEBUG sleep 30

此命令将使我们的主服务器无法访问，休眠30秒。
它基本上模拟了主服务器因某种原因无响应。

如果您查看Sentinel日志，应该能够看到很多活动：

1. 每个哨兵都会通过`+sdown`事件检测到主服务器宕机。
2. 这个事件随后升级为`+odown`，意味着多个哨兵一致认为主服务器无法访问。
3. 哨兵们会投票选出一个哨兵来启动第一次故障切换尝试。
4. 故障切换发生。

如果你再次询问`mymaster`的当前主地址，最终这次我们应该会得到不同的回答：

    127.0.0.1:5000> SENTINEL get-master-addr-by-name mymaster
    1) "127.0.0.1"
    2) "6380"

到目前为止一切顺利... 在这一点上，您可以立即跳转至创建您的 Sentinel 部署，
或者可以阅读更多内容来了解所有 Sentinel 命令和内部机制。

## Sentinel API

Sentinel提供API以便检查其状态，检查被监视的主服务器和从服务器的健康情况，订阅以接收特定通知，并在运行时更改Sentinel配置。

默认情况下，Sentinel使用TCP端口26379（请注意，6379是正常的Redis端口）。Sentinel接受使用Redis协议的命令，因此您可以使用`redis-cli`或任何其他未修改的Redis客户端来与Sentinel进行通信。

有可能直接查询一个Sentinel，以检查从它的角度来看，被监视的Redis实例的状态，了解它知道的其他Sentinel等等。或者，使用Pub/Sub，可以接收到来自Sentinel的"推送式"通知，每当发生某些事件时，例如故障切换、实例进入错误状态等等。

### Sentinel 命令列表

`SENTINEL` 命令是 Sentinel 的主 API。以下是它的子命令列表（适用时已注明最低版本）：

* **SENTINEL CONFIG GET `<name>`** (`>= 6.2`) 获取全局 Sentinel 配置参数的当前值。指定的名称可以使用通配符，类似于 Redis 的 `CONFIG GET` 命令。
* **SENTINEL CONFIG SET `<name>` `<value>`** (`>= 6.2`) 设置全局 Sentinel 配置参数的值。
* **SENTINEL CKQUORUM `<master name>`** 检查当前 Sentinel 配置是否能够达到进行主节点故障迁移所需的法定人数，并且达到授权故障迁移所需的多数票数。此命令应在监控系统中使用，以检查 Sentinel 部署是否正常。
* **SENTINEL FLUSHCONFIG** 强制 Sentinel 在磁盘上重写其配置，包括当前 Sentinel 状态。通常情况下，Sentinel 每次状态发生变化时都会重写配置（在跨重启时磁盘上持久化的状态子集的上下文中）。然而，有时由于操作错误、磁盘故障、软件包升级脚本或配置管理器导致配置文件丢失。在这些情况下，强制 Sentinel 重写配置文件是很方便的。即使之前的配置文件完全丢失，此命令也能正常工作。
* **SENTINEL FAILOVER `<master name>`** 强制进行故障迁移，就好像主节点不可达一样，并且不需要其他 Sentinel 的同意（但将发布一个新版本的配置，以便其他 Sentinel 更新其配置）。
* **SENTINEL GET-MASTER-ADDR-BY-NAME `<master name>`** 返回具有该名称的主节点的 IP 地址和端口号。如果此主节点正在进行故障迁移或正在成功完成故障迁移，则返回晋升的从节点的地址和端口。
* **SENTINEL INFO-CACHE** (`>= 3.2`) 返回主节点和从节点的缓存 `INFO` 输出。
* **SENTINEL IS-MASTER-DOWN-BY-ADDR <ip> <port> <current-epoch> <runid>** 检查由 ip:port 指定的主节点是否从当前 Sentinel 的视角下线。此命令主要用于内部使用。
* **SENTINEL MASTER `<master name>`** 显示指定主节点的状态和信息。
* **SENTINEL MASTERS** 显示监视的主节点列表及其状态。
* **SENTINEL MONITOR** 启动 Sentinel 的监控。有关更多信息，请参阅[_运行时重新配置 Sentinel_ 部分](#reconfiguring-sentinel-at-runtime)。
* **SENTINEL MYID** (`>= 6.2`) 返回 Sentinel 实例的 ID。
* **SENTINEL PENDING-SCRIPTS** 此命令返回有关待处理脚本的信息。
* **SENTINEL REMOVE** 停止 Sentinel 的监控。有关更多信息，请参阅[_运行时重新配置 Sentinel_ 部分](#reconfiguring-sentinel-at-runtime)。
* **SENTINEL REPLICAS `<master name>`** (`>= 5.0`) 显示此主节点的从节点列表及其状态。
* **SENTINEL SENTINELS `<master name>`** 显示此主节点的 Sentinel 实例列表及其状态。
* **SENTINEL SET** 设置 Sentinel 的监控配置。有关更多信息，请参阅[_运行时重新配置 Sentinel_ 部分](#reconfiguring-sentinel-at-runtime)。
* **SENTINEL SIMULATE-FAILURE (crash-after-election|crash-after-promotion|help)** (`>= 3.2`) 此命令模拟不同的 Sentinel 崩溃场景。
* **SENTINEL RESET `<pattern>`** 此命令将重置与名称匹配的所有主节点。模式参数是一个类似于 glob 的模式。重置过程会清除主节点中的任何先前状态（包括正在进行的故障迁移），并删除已发现并与主节点关联的所有从节点和 Sentinel。

为了连接管理和管理目的，Sentinel支持Redis以下子集的命令：


* **ACL** (`>= 6.2`) 此指令管理 Sentinel 访问控制列表。有关更多信息，请参阅 [ACL](/topics/acl) 文件和 [_Sentinel 访问控制列表身份验证_](#sentinel-access-control-list-authentication)。
* **AUTH** (`>= 5.0.1`) 对客户端连接进行身份验证。有关更多信息，请参阅 `AUTH` 指令和 [_使用身份验证配置 Sentinel 实例_ 部分](#configuring-sentinel-instances-with-authentication)。
* **CLIENT** 此指令管理客户端连接。有关更多信息，请参阅其子指令的页面。
* **COMMAND** (`>= 6.2`) 此指令返回有关指令的信息。有关更多信息，请参阅 `COMMAND` 指令及其各种子指令。
* **HELLO** (`>= 6.0`) 切换连接的协议。有关更多信息，请参阅 `HELLO` 指令。
* **INFO** 返回有关 Sentinel 服务器的信息和统计数据。有关更多信息，请参阅 `INFO` 指令。
* **PING** 此指令仅返回 PONG。
* **ROLE** 此指令返回字符串 "sentinel" 和一组被监视的主服务器。有关更多信息，请参阅 `ROLE` 指令。
* **SHUTDOWN** 关闭 Sentinel 实例。

最后，Sentinel还支持`SUBSCRIBE`、`UNSUBSCRIBE`、`PSUBSCRIBE`和`PUNSUBSCRIBE`命令。更多详情请参考[_发布/订阅消息_](#pubsub-messages)部分。

### 在运行时重新配置 Sentinel

从Redis版本2.8.4开始，Sentinel提供了一个API，用于添加、删除或更改特定主服务器的配置。请注意，如果您有多个Sentinel实例，应将更改应用到所有实例上，以使Redis Sentinel正常工作。这意味着更改单个Sentinel的配置不会自动传播到网络中的其他Sentinel实例。

以下是用于更新Sentinel实例配置的“SENTINEL”子命令列表。

* **SENTINEL MONITOR `<name>` `<ip>` `<port>` `<quorum>`** 这个命令告诉 Sentinel 开始监视一个新的主服务器，指定了名称、IP、端口和投票数。它与 `sentinel.conf` 配置文件中的 `sentinel monitor` 配置指令完全相同，唯一的区别是你不能使用主机名作为 `ip`，而是需要提供 IPv4 或 IPv6 地址。
* **SENTINEL REMOVE `<name>`** 用于从 Sentinel 内部状态中移除指定的主服务器：该主服务器将不再被监视，并且完全从 Sentinel 中移除，因此不再被 `SENTINEL masters` 等命令列出。
* **SENTINEL SET `<name>` [`<option>` `<value>` ...]** SET 命令与 Redis 的 `CONFIG SET` 命令非常相似，用于更改特定主服务器的配置参数。可以指定多个选项/值对（或者不指定任何选项/值）。所有可以通过 `sentinel.conf` 配置的配置参数也可以使用 SET 命令进行配置。

以下是`SENTINEL SET`命令的示例，用于修改名为`objects-cache`的主节点的`down-after-milliseconds`配置：

    SENTINEL SET objects-cache-master down-after-milliseconds 1000

正如已经提到的，`SENTINEL SET` 可用于设置启动配置文件中可以设置的所有配置参数。此外，还可以仅更改主服务器仲裁配置，而无需使用 `SENTINEL REMOVE` 后跟 `SENTINEL MONITOR` 来删除和重新添加主服务器，而是简单地使用：

    SENTINEL SET objects-cache-master quorum 5

请注意，由于`SENTINEL MASTER`以易于解析的格式（作为字段/值对数组）提供了所有配置参数，所以没有等效的GET命令。

Redis 从版本 6.2 开始，Sentinel 也支持获取和设置全局配置参数，这些参数在此之前仅支持通过配置文件设置。

* **SENTINEL CONFIG GET `<name>`** 获取全局Sentinel配置参数的当前值。指定的名称可以是通配符，类似于Redis的“CONFIG GET”命令。
* **SENTINEL CONFIG SET `<name>` `<value>`** 设置全局Sentinel配置参数的值。

可以操作的全局参数包括：

```
* `resolve-hostnames`, `announce-hostnames`. 查看[_IP 地址和 DNS 名字_](#ip-addresses-and-dns-names)。
* `announce-ip`, `announce-port`. 查看[_Sentinel, Docker, NAT 以及可能出现的问题_](#sentinel-docker-nat-and-possible-issues)。
* `sentinel-user`, `sentinel-pass`. 查看[_配置带有身份验证的 Sentinel 实例_](#configuring-sentinel-instances-with-authentication)。
```

### 添加或删除哨兵

由于Sentinel实施了自动发现机制，因此将新的Sentinel添加到您的部署中是一个简单的过程。您需要做的就是启动新的Sentinel，并配置为监视当前活动的主节点。在10秒内，Sentinel将获得其他Sentinel的列表以及附加到主节点的副本集。

如果您需要一次添加多个Sentinel，请建议将其添加“一个接一个”，在添加下一个之前等待所有其他Sentinel已知
关于第一个Sentinel。在添加新的Sentinel的过程中，这样做有助于仍然确保在分区的一侧仅能实现大多数，
这样一来，即使在添加新的Sentinel的过程中出现故障，也能保证其有效性。

这可以通过在每个新 Sentinel 上添加30秒的延迟来轻松实现，并且在网络分区期间。

在过程结束时，可以使用命令`SENTINEL MASTER mastername`来检查所有 Sentinel 是否一致同意监视主服务器的 Sentinel 总数。

删除哨兵有点复杂：哨兵永远不会忘记已经看到的哨兵，即使它们长时间不可达，因为我们不希望动态改变授权故障转移和创建新配置号所需的大多数。因此，为了删除一个哨兵，在不存在网络分区的情况下，应执行以下步骤：

1. 停止要删除的Sentinel进程。
2. 向所有其他Sentinel实例发送`SENTINEL RESET *`命令（如果只想重置一个主节点，可以使用确切的主节点名称替代`*`）。 一个接一个地进行，每个实例之间至少等待30秒。
3. 通过检查每个Sentinel的`SENTINEL MASTER mastername`输出，确保所有Sentinel都同意当前活动的Sentinel数量。

###删除旧主服务器或无法访问的副本

# Sentinels永远不会忘记给定主节点的复制品，即使它们长时间无法访问。 这是有用的，因为Sentinels应该能够在网络分区或故障事件之后正确地重新配置返回的复制品。

此外，故障切换后，故障切换主节点将被虚拟添加为新主节点的副本，这样它将被重新配置为在新主节点可用时进行复制。

有时您可能希望从由Sentinel监视的副本列表中永久删除一个副本（可能是旧主机）。

为了做到这一点，您需要向所有Sentinel发送`SENTINEL RESET mastername`命令，
它们将在接下来的10秒内刷新副本列表，只添加在当前主节点`INFO`输出中被正确复制的副本。

### 发布/订阅消息

客户端可以使用 Sentinel 作为与 Redis 兼容的发布/订阅服务器（但不能使用 `PUBLISH`）来 `SUBSCRIBE` 或 `PSUBSCRIBE` 到频道，并在特定事件发生时得到通知。

频道名称与事件名称相同。例如，名为`+sdown`的频道将接收与实例进入`SDOWN`（SDOWN表示从您查询的Sentinel的角度来看，实例不再可达）状态相关的所有通知。

为了获取所有的消息，只需使用`PSUBSCRIBE *`订阅即可。

以下是您可以使用此API接收的频道和消息格式的列表。第一个单词是频道/事件名称，其余是数据的格式。

注意：在指定 *实例详情* 时，意味着以下参数提供用于标识目标实例：

    <instance-type> <name> <ip> <port> @ <master-name> <master-ip> <master-port>

标识主服务器的部分（从 @ 参数到末尾）是可选的，仅在该实例不是主服务器本身时才会指定。

* **+reset-master** `<instance details>` -- 主节点已重置。
* **+slave** `<instance details>` -- 检测到新的副本并连接。
* **+failover-state-reconf-slaves** `<instance details>` -- 切换状态变为`reconf-slaves`状态。
* **+failover-detected** `<instance details>` -- 检测到另一个Sentinel或其他外部实体启动的故障转移（一个连接的副本变为主节点）。
* **+slave-reconf-sent** `<instance details>` -- 主Sentinel向该实例发送`REPLICAOF`命令，以重新配置其为新的副本。
* **+slave-reconf-inprog** `<instance details>` -- 正在重新配置的副本显示为新主的IP:端口对的副本，但同步过程尚未完成。
* **+slave-reconf-done** `<instance details>` -- 副本现已与新的主节点同步。
* **-dup-sentinel** `<instance details>` -- 已删除指定主节点的一个或多个重复的Sentinel（例如，当重启一个Sentinel实例时会发生此情况）。
* **+sentinel** `<instance details>` -- 检测到为此主节点添加了新的Sentinel并连接。
* **+sdown** `<instance details>` -- 指定的实例现在处于”主观下线“状态。
* **-sdown** `<instance details>` -- 指定的实例不再处于”主观下线“状态。
* **+odown** `<instance details>` -- 指定的实例现在处于“客观下线”状态。
* **-odown** `<instance details>` -- 指定的实例不再处于“客观下线”状态。
* **+new-epoch** `<instance details>` -- 当前纪元已更新。
* **+try-failover** `<instance details>` -- 有新的故障转移正在进行中，等待大多数选举。
* **+elected-leader** `<instance details>` -- 在指定的纪元中赢得选举，可以进行故障转移。
* **+failover-state-select-slave** `<instance details>` -- 新的故障转移状态为`select-slave`：我们正在尝试找到一个合适的副本进行升级。
* **no-good-slave** `<instance details>` -- 没有合适的副本可升级。目前我们会在一段时间后尝试，但很可能会更改，而在这种情况下，状态机将完全中止故障转移。
* **selected-slave** `<instance details>` -- 我们找到了指定的合适的副本进行升级。
* **failover-state-send-slaveof-noone** `<instance details>` -- 我们正在尝试将升级后的副本重新配置为主节点，等待其切换。
* **failover-end-for-timeout** `<instance details>` -- 由于超时，故障转移终止，副本最终将配置为与新的主节点进行复制。
* **failover-end** `<instance details>` -- 故障转移成功终止。所有副本似乎已经重新配置为与新的主节点进行复制。
* **switch-master** `<master name> <oldip> <oldport> <newip> <newport>` -- 在配置更改后，主节点的新IP和地址为指定的地址。这是**大多数外部用户感兴趣的信息**。
* **+tilt** -- 进入Tilt模式。
* **-tilt** -- 退出Tilt模式。

### -BUSY 状态的处理

在配置的Lua脚本时间限制内，当Redis实例执行Lua脚本的时间超过限制时，会返回-BUSY错误。发生这种情况时，在触发故障切换之前，Redis Sentinel将尝试发送“SCRIPT KILL”命令，只有在脚本是只读的情况下才能成功。

如果实例在此尝试之后仍然处于错误状态，最终将进行故障切换。

优先级 复制品

Redis实例有一个名为`replica-priority`的配置参数。此信息在Redis复制实例的`INFO`输出中公开，Sentinel使用它来选择一个可用于故障切换主服务器的副本。

1. 如果将复制品的优先级设置为0，则该复制品永远不会被提升为主要副本。
2. Sentinel更倾向于具有较低优先级编号的复制品。

例如，如果在当前主服务器所在的数据中心中存在一个副本 S1，并且在另一个数据中心中存在另一个副本 S2，则可以将 S1 的优先级设置为10，将 S2 的优先级设置为100，以便在主服务器故障且 S1 和 S2 都可用时，优先选择 S1。

有关副本的选择方式的更多信息，请查看本文档的[_副本选择和优先级_部分](#replica-selection-and-priority)。

### Sentinel 和 Redis 认证

当主机配置为要求客户端进行身份验证时，作为一项安全措施，副本也需要知道凭据以便与主机进行身份验证，并创建用于异步复制协议的主副本连接。

## Redis 访问控制列表身份验证

从Redis 6开始，用户身份验证和权限管理由 [访问控制列表 (ACL)](/topics/acl) 来管理。

为了让哨兵在配置了访问控制列表（ACL）的情况下连接到Redis服务器实例，哨兵配置必须包括以下指令：

    sentinel auth-user <master-name> <username>
    sentinel auth-pass <master-name> <password>

当`<username>`和`<password>`是访问该组实例的用户名和密码时。这些凭证应该在该组的所有Redis实例上配置，并具备最小的控制权限。例如：

    127.0.0.1:6379> ACL SETUSER sentinel-user ON >somepassword allchannels +multi +slaveof +ping +exec +subscribe +config|rewrite +role +publish +info +client|setname +client|kill +script|kill

### Redis仅密码验证

在Redis 6之前，通过以下配置指令实现身份验证：

* 在主服务器中使用`requirepass`命令设置身份验证密码，确保实例不会处理未经身份验证的客户端请求。
* 在从服务器中使用`masterauth`命令使从服务器能够通过身份验证与主服务器进行通信，从而正确地复制数据。

当使用Sentinel时，并没有单个主节点，因为在故障切换后，从节点可以扮演主节点的角色，而旧的主节点可以被重新配置为从节点，所以您需要在所有实例中设置上述指令，无论是主节点还是从节点。

这通常也是一个理智的设置，因为您不希望仅在主数据库中保护数据，副本中的数据也应该可以访问。

然而，如果您需要一个无需身份验证即可访问的副本的情况很少见，您仍然可以通过设置**优先级为零**的副本来实现，以防止该副本被提升为主服务器，并且仅在该副本中配置`masterauth`指令，而不是使用`requirepass`指令，以便未经身份验证的客户端可以读取数据。

为了让Sentinel与配置了`requirepass`的Redis服务器实例建立连接，Sentinel配置必须包括`sentinel auth-pass`指示符，格式如下：

    sentinel auth-pass <master-name> <password>

使用身份验证配置Sentinel实例
---


Sentinel实例本身可以通过要求客户端通过`AUTH`命令进行身份验证来进行安全保护。从Redis 6.2开始，可以使用[访问控制列表（ACL）](/topics/acl)，而之前的版本（从Redis 5.0.1开始）仅支持密码身份验证。

请注意，Sentinel的身份验证配置应**应用于部署中的每个实例**，**所有实例应使用相同的配置**。此外，ACL和仅密码验证不能同时使用。

### Sentinel访问控制列表身份验证

使用ACL保护Sentinel实例的第一步是阻止未经授权的访问。为了做到这一点，您需要禁用默认的超级用户（或者至少设置一个强密码），然后创建一个新的超级用户并允许其访问Pub/Sub通道：

    127.0.0.1:5000> ACL SETUSER admin ON >admin-password allchannels +@all
    OK
    127.0.0.1:5000> ACL SETUSER default off
    OK

# 默认用户是Sentinel用来连接其他实例的用户。您可以通过以下配置指令提供另一个超级用户的凭证：

    sentinel sentinel-user <username>
    sentinel sentinel-pass <password>

其中`<username>`和`<password>`分别是Sentinel的超级用户和密码（例如上面示例中的`admin`和`admin-password`）。

最后，为了对传入的客户端连接进行身份验证，您可以创建一个仅限 Sentinel 用户配置文件，如下所示：

    127.0.0.1:5000> ACL SETUSER sentinel-user ON >user-password -@all +auth +client|getname +client|id +client|setname +command +hello +ping +role +sentinel|get-master-addr-by-name +sentinel|master +sentinel|myid +sentinel|replicas +sentinel|sentinels

请参考您选择的Sentinel客户端的文档以获取更多信息。

### Sentinel仅密码验证

要使用仅密码身份验证的Sentinel，请按照以下方式将`requirepass`配置指令添加到**所有**Sentinel实例中：

    requirepass "your_password_here"

当以这种方式配置时，Sentinels将执行两个操作：

1. 为了向 Sentinel 发送命令，客户端需要提供密码。这是因为这在 Redis 中是配置指令的通常方式。
2. 此外，用于访问本地 Sentinel 的相同密码，将被此 Sentinel 实例用于向其连接的所有其他 Sentinel 实例进行身份验证。

这意味着**您必须在所有哨兵实例中配置相同的`requirepass`密码**。这样每个哨兵都可以与其他哨兵进行通信，而无需为每个哨兵配置访问所有其他哨兵的密码，这将非常不方便。

使用此配置之前，请确保您的客户端库可以向 Sentinel 实例发送 `AUTH` 命令。

### Sentinel客户端实现

除非系统配置为执行一个脚本，该脚本将所有请求透明重定向到新主实例（虚拟IP或其他类似系统），否则Sentinel需要明确的客户端支持。有关客户端库的实现主题，请参阅《Sentinel客户端指南》文档(/topics/sentinel-clients)。

## 更高级的概念

在接下来的几个部分中，我们将介绍Sentinel工作的一些细节，而无需涉及本文档最后部分将涵盖的实现细节和算法。

### SDOWN 和 ODOWN 失效状态

Redis Sentinel有两种不同的“宕机”概念，一种称为“主观宕机”(SDOWN)，是针对特定Sentinel实例的宕机状态。另一种是称为“客观宕机”(ODOWN)，当足够多的Sentinel（至少配置为受监视的主节点的`quorum`参数数量）具有SDOWN状态，并通过使用`SENTINEL is-master-down-by-addr`命令从其他Sentinel获得反馈时，就会达到ODOWN状态。

从Sentinel的角度来看，当它在配置中指定的“is-master-down-after-milliseconds”参数中未收到PING请求的有效回复的时间达到一定秒数时，将达到SDOWN状态。

PING 的可接受回复应为以下之一：

* PING回复了+PONG。
* PING回复了-LOADING错误。
* PING回复了-MASTERDOWN错误。

任何其他回复（或根本没有回复）均被视为无效。
但请注意，INFO 输出中自称为副本的逻辑主节点被视为宕机。

请注意，SDOWN要求整个配置的时间间隔内没有接收到可接受的回复。例如，如果时间间隔为30000毫秒（30秒），并且我们每29秒收到一个可接受的ping回复，那么该实例被认为是正常工作的。

`SDOWN`不足以触发故障转移：它仅表示一个Sentinel认为Redis实例不可用。要触发故障转移，必须达到`ODOWN`状态。

为了从SDOWN切换到ODOWN，不使用强一致性算法，而是使用一种形式的流言传播：如果一个给定的Sentinel从足够多的Sentinels那里收到了主节点在给定时间范围内不工作的报告，那么SDOWN会被提升为ODOWN。如果这个确认稍后消失了，那么标志会被清除。

需要更严格的授权才能实际启动故障转移，但必须达到 ODOWN 状态才能触发故障转移。

ODOWN条件**仅适用于主实例**。对于其他类型的实例，Sentinel不需要采取行动，因此副本和其他Sentinel永远不会达到ODOWN状态，只会达到SDOWN状态。

然而，SDOWN也具有语义含义。例如，处于SDOWN状态的副本不会被Sentinel选中并提升进行故障转移。

哨兵和副本自动发现

Sentinel 通过与其他 Sentinel 保持连接，以相互检查彼此的可用性并交换消息。但是，您不需要在每个运行的 Sentinel 实例中配置其他 Sentinel 地址列表，因为 Sentinel 使用 Redis 实例的发布/订阅功能来发现监视相同主服务器和从服务器的其他 Sentinel。

这个功能是通过将“hello消息”发送到名为`__sentinel__:hello`的频道来实现的。

同样，您不需要配置主服务器附加的副本列表，因为Sentinel会自动发现此列表并查询Redis。

* 每个 Sentinel 每两秒钟向每个监视的主节点和副本 Pub/Sub 频道 `__sentinel__:hello` 发布一条消息，以ip、端口、运行id宣告其存在。
* 每个 Sentinel 都订阅每个主节点和副本的 Pub/Sub 频道 `__sentinel__:hello` ，以查找未知的 Sentinel。当检测到新的 Sentinel 时，将其添加为该主节点的 Sentinel。
* Hello 消息还包括主节点的当前完整配置。如果接收到的 Sentinel 的给定主节点配置比接收到的旧配置要旧，它会立即更新到新配置。
* 在向主节点添加新的 Sentinel 之前，Sentinel 总是检查是否已经存在具有相同运行id或相同地址（ip和端口对）的 Sentinel。在这种情况下，所有匹配的 Sentinel 都将被删除，并添加新的 Sentinel。

在故障切换过程之外重新配置Sentinel实例

即使没有故障转移正在进行，Sentinel 仍会尝试在被监视的实例上设置当前配置。具体地说：

* 从当前配置中，声称自己是主服务器的副本，将被配置为副本，以与当前的主服务器进行复制。
* 连接到错误主服务器的副本，将被重新配置为与正确的主服务器进行复制。

要让Sentinels重新配置副本，必须观察到错误配置的时间超过用于广播新配置的周期。

这样可以防止具有陈旧配置的Sentinels（例如因为它们刚从分区重新加入）在接收更新之前尝试更改副本配置。

同时也要注意到，始终试图强制当前配置的语义使故障转移对分区更具抵抗力：

* 主服务器在发生故障切换后，当恢复可用时，会重新配置为副本。
* 当副本在分区期间被分隔出去后，一旦恢复连接，会重新配置。

关于这一部分要记住的重要教训是：**Sentinel 是一个系统，其中每个进程都会始终尝试将最后一个逻辑配置强加给一组被监视的实例**。

### 复制品选择和优先级

当Sentinel实例准备执行故障转移时，由于主节点处于`ODOWN`状态，并且Sentinel从大多数已知的Sentinel实例中收到了进行故障转移的授权，需要选择一个合适的副本。

副本选择过程评估了关于副本的以下信息：

1. 从主节点断开的时间。
2. 复制优先级。
3. 复制偏移量已处理。
4. 运行 ID。

如果发现主节点与备份节点断开连接的时间超过配置的主节点超时时间（down-after-milliseconds选项）的十倍以上，再加上在Sentinel进行故障转移时主节点不可用的时间，该备份节点被视为不适合进行故障转移而被跳过。

更严格地说，一台副本的`INFO`输出表明它已与主服务器断开连接超过：

    (down-after-milliseconds * 10) + milliseconds_since_master_is_in_SDOWN_state

被认为不可靠，并完全被忽视。

复制品选择仅考虑通过上述测试的副本，并根据上述标准按以下顺序对其进行排序。

1. 副本按照在 Redis 实例的 `redis.conf` 文件中配置的 `replica-priority` 进行排序。较低的优先级将被优先选择。
2. 如果优先级相同，则检查副本处理的复制偏移量，选择从主节点接收更多数据的副本。
3. 如果多个副本具有相同的优先级并且处理了来自主节点相同的数据，则进行进一步检查，选择字典顺序较小的运行 ID。拥有较低的运行 ID 对副本来说并不是真正的优势，但它有助于使副本选择过程更具确定性，而不是选择一个随机的副本。

在大多数情况下，`replica-priority`不需要被显式设置，因此所有实例将使用相同的默认值。如果有特定的故障切换优先级，必须在所有实例上设置`replica-priority`，包括主实例，因为主实例在将来的某个时间点可能会变为副本，这时它将需要适当的`replica-priority`设置。

一个 Redis 实例可以通过将特殊的 `replica-priority` 设置为零来配置，这样它就**永远不会被** Sentinel 选为新的主节点。
然而，以这种方式配置的复制节点在故障切换后仍会被 Sentinel 重新配置以与新的主节点进行复制，唯一的区别是它永远不会成为主节点。

## 算法和内部结构

在接下来的部分，我们将探讨 Sentinel 行为的详细信息。
用户并不一定需要了解所有的细节，但对 Sentinel 的深入理解可能有助于更有效地部署和操作 Sentinel。

### 法定人数

以前的章节显示，由Sentinel监视的每个主节点都与一个配置的**投票数（quorum）**相关联。它指定了需要达成一致意见的Sentinel进程数量，以便在主节点不可访问或出现错误的情况下触发故障转移。

然而，一旦触发故障转移，在实际执行故障转移之前，**至少大多数 Sentinel 必须授权 Sentinel 执行故障转移**。当只有少数 Sentinel 存在于一个分区上时，Sentinel 永远不会执行故障转移。

让我们试着把事情弄得更清楚一些：

* **Quorum**: Sentinel进程需要检测到错误条件的数量，以便将主服务器标记为**ODOWN**。
* 失败转移由**ODOWN**状态触发。
* 一旦触发了故障转移，试图执行故障转移的Sentinel需要向大多数Sentinel请求授权（或者如果quorum设置为大于多数的数字，则需要超过多数的授权）。

看起来差异微小，但实际上非常容易理解和使用。例如，如果你有5个Sentinel实例，并且仲裁被设置为2，一旦有2个Sentinel认为主服务器无法访问，故障转移就会被触发，但是仅有其中的一个Sentinel能够执行故障转移，只有当它至少从3个Sentinel获得授权。

如果配置了5个法定人数，那么所有的Sentinel节点必须就主节点错误的情况达成一致，并且需要所有Sentinel节点的授权才能进行故障切换。

这意味着法定人数可以用两种方式来调节哨兵：

1. 如果将法定人数设置为小于我们部署的Sentinel大多数的值，我们基本上是使Sentinel对主节点故障更加敏感，只要Sentinel的少数人不能再与主节点通信，就会触发故障转移。
2. 如果将法定人数设置为大于Sentinel大多数的值，我们只有在有大量（超过多数）与主节点通信良好的Sentinel都同意主节点已停止运行时，才能使Sentinel进行故障转移。

### 配置时期

哨兵需要从大多数途径获得授权，以便开始故障转移，原因有以下几点：

当授权了一个 Sentinel 时，它会得到一个独特的 **配置纪元**，用于进行其正在进行故障切换的主服务器的配置更新。这个数字将用于在故障切换完成后对新配置进行版本控制。因为多数 Sentinel 都同意某个版本分配给了某个 Sentinel，其他 Sentinel 就无法使用它。这意味着每次故障切换的配置都会有一个独特的版本。我们将会看到为什么这一点非常重要。

此外，Sentinel还有一个规则：如果一个Sentinel为给定主服务器的故障转移投票给另一个Sentinel，它会等待一段时间再次尝试对同一主服务器进行故障转移。这个延迟是在 `sentinel.conf` 中可以配置的`2 * failover-timeout`。这意味着Sentinel不会同时尝试对同一主服务器进行故障转移，第一个请求授权的Sentinel将尝试，如果失败，之后会有另一个Sentinel在一段时间后再次尝试，以此类推。

Redis Sentinel保证了“liveness”属性，即如果大多数Sentinels可以进行通信，最终其中一个将被授权在主服务器宕机时执行故障切换。

Redis Sentinel还保证了每个Sentinel都会使用不同的*配置时期*故障转移相同的主节点的*safety*属性。

### 配置传播

一旦 Sentinel 成功地进行主节点故障转移，它将开始广播新的配置，以便其他 Sentinel 更新关于给定主节点的信息。

为了使故障转移被视为成功，需要Sentinel能够将`REPLICAOF NO ONE`命令发送给所选的从节点，并且在主节点的`INFO`输出中观察到主节点切换后。

> 在这个阶段，即

新配置被传播的方式是为什么我们需要每次Sentinel故障转移都得授权一个不同的版本号（配置时期）的原因。

每个哨兵都使用Redis Pub/Sub消息广播其关于主服务器配置的版本，无论是在主服务器还是所有副本中。同时，所有哨兵都在等待消息以查看其他哨兵所广告的配置。

配置以 `__sentinel__:hello` 的方式通过发布/订阅通道进行广播。

由于每个配置都有不同的版本号，较大的版本始终优先于较小的版本。

例如，主节点`mymaster`的配置以所有Sentinels都认为主节点在192.168.1.50:6379上开始。此配置的版本为1。经过一段时间，一个Sentinel被授权用版本2进行故障转移。如果故障转移成功，它将开始广播一个新的配置，比如说192.168.1.50:9000，并且版本号为2。所有其他实例将看到这个配置并相应地更新自己的配置，因为新的配置具有更高的版本号。

这意味着Sentinel会保证第二个活性属性：能够通信的Sentinel集合将会收敛到具有较高版本号的相同配置。

基本上，如果网络被划分，每个分区将会收敛到较高的本地配置。在没有分区的特殊情况下，有一个单一分区，每个 Sentinel 都将对配置达成一致。

### 分区下的一致性

Redis Sentinel配置最终一致，因此每个分区将会收敛到最高可用配置。
然而，在使用Sentinel的真实系统中，存在三种不同的角色：

* Redis 实例。
* Sentinel 实例。
* 客户端。

为了定义系统的行为，我们必须考虑所有三个因素。

以下是一个简单的网络，其中有3个节点，每个节点运行一个Redis实例和一个Sentinel实例：

                +-------------+
                | Sentinel 1  |----- Client A
                | Redis 1 (M) |
                +-------------+
                        |
                        |
    +-------------+     |          +------------+
    | Sentinel 2  |-----+-- // ----| Sentinel 3 |----- Client B
    | Redis 2 (S) |                | Redis 3 (M)|
    +-------------+                +------------+

在这个系统中，初始状态是Redis 3作为主服务器，而Redis 1和2作为副本服务器。发生了一个分区，使旧的主服务器被隔离。Sentinels 1和2启动了故障转移，将Sentinel 1提升为新的主服务器。

保证Sentinel 1和Sentinel 2的属性具有新的主配置。然而，由于Sentinel 3位于不同的分区中，因此仍然具有旧的配置。

我们知道当网络分区恢复时，Sentinel 3将获得配置更新，但是在分区期间，如果有客户端被分区并与旧主节点分区会发生什么？

客户端仍然能够写入Redis 3，即旧的主节点。当分区重新加入时，Redis 3将变成Redis 1的副本，分区期间写入的所有数据将丢失。

根据您的配置，您可能希望或不希望发生这种情况：

* 如果您将Redis用作缓存，即使Client B的数据将会丢失，它仍然可以继续写入旧主节点，这是非常方便的。
* 如果您将Redis用作存储，那么这种情况就不好了，您需要配置系统以部分地防止这个问题的发生。

由于Redis是异步复制的，因此在这种情况下无法完全防止数据丢失，但是您可以使用以下Redis配置选项限定Redis 3和Redis 1之间的差异。

    min-replicas-to-write 1
    min-replicas-max-lag 10

使用上述配置（请参见Redis分发中的自注释的`redis.conf`示例以获取更多信息），当Redis实例充当主机时，如果无法写入至少1个副本，它将停止接受写入。由于复制是异步的，*无法写入*实际上意味着副本要么断开连接，要么在指定的`max-lag`秒内没有发送异步确认信息给我们。

使用这个配置后，上面例子中的 Redis 3 将在 10 秒后变得不可用。当分区恢复时，Sentinel 3 配置将会收敛到新配置，并且客户端 B 将能够获取有效的配置并继续执行。

通常情况下，Redis + Sentinel作为一个整体被视为一个**最终一致性系统**，其中合并函数是**最后一次故障转移胜出**，而来自旧主节点的数据被丢弃以复制当前主节点的数据，因此总会有一段时间可能丢失已确认写入的窗口。这是由于Redis异步复制和系统的“虚拟”合并函数的丢弃性质所致。请注意，这不是Sentinel本身的限制，如果您使用强一致性的复制状态机编排故障转移，这些属性仍然适用。只有两种方法可避免丢失已确认的写入：

## 使用同步复制（以及适当的共识算法来运行复制的状态机）。

## 使用最终一致的系统，可以合并同一对象的不同版本。

Redis当前无法使用上述系统之一，目前也不在开发目标之列。然而，有些代理实现了在Redis存储之上的“2”解决方案，例如SoundCloud的[Roshi](https://github.com/soundcloud/roshi)或Netflix的[Dynomite](https://github.com/Netflix/dynomite)。

哨兵持久化状态
---

Sentinel状态保存在sentinel配置文件中。例如，每次接收到新的配置或创建新的配置（领导者Sentinels）时，对于一个主节点，配置将与配置纪元一起保存在磁盘上。这意味着可以安全地停止和重新启动Sentinel进程。

### 倾斜模式

Redis Sentinel非常依赖计算机时间：例如，为了确定一个实例是否可用，它会记住PING命令的最新成功响应时间，并与当前时间进行比较以确定它的年龄。

然而，如果计算机时间以意外的方式改变，或者计算机非常繁忙，或者出于某种原因进程被阻塞，Sentinel可能会表现出意想不到的方式。

TILT模式是指当Sentinel检测到可能降低系统可靠性的异常情况时，可以进入的特殊“保护”模式。
Sentinel定时中断通常每秒调用10次，因此我们预计在两次定时器中断调用之间会经过大约100毫秒的时间。

Sentinel的作用是记录上一次定时器中断被调用的时间，并将其与当前调用进行比较：如果时间差为负或者异常大（2秒或更多），则进入倾斜模式（或者如果已经进入了倾斜模式，则推迟退出倾斜模式）。

在TILT模式下，Sentinel将继续监视一切，但是：

* 它完全停止工作。
* 它开始对 `SENTINEL is-master-down-by-addr` 的请求作出负面回应，因为不再信任检测故障的能力。

如果一切正常持续30秒，退出TILT模式。
 
在Sentinel倾斜模式下，如果我们发送INFO命令，我们可以得到以下响应：

    $ redis-cli -p 26379
    127.0.0.1:26379> info
    (Other information from Sentinel server skipped.)

    # Sentinel
    sentinel_masters:1
    sentinel_tilt:0
    sentinel_tilt_since_seconds:-1
    sentinel_running_scripts:0
    sentinel_scripts_queue_length:0
    sentinel_simulate_failure_flags:0
    master0:name=mymaster,status=ok,address=127.0.0.1:6379,slaves=0,sentinels=1

字段"sentinel_tilt_since_seconds"表示Sentinel已经处于TILT模式的秒数。
如果它不在TILT模式下，则值为-1。

请注意，在某种程度上，TILT模式可以使用许多内核提供的单调时钟API进行替代。然而，目前还不清楚这是否是一个好的解决方案，因为当前系统避免了进程长时间被挂起或未被调度器执行的问题。

**关于在此手册中使用“slave”一词的说明**：从Redis 5开始，如果不考虑向后兼容性，Redis项目不再使用“slave”一词。不幸的是，在此命令中，“slave”一词是协议的一部分，所以只有在此API被自然弃用时，我们才能删除这类出现。
