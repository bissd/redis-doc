---
title: "安装Redis"
linkTitle: "安装Redis"
weight: 1
description: >
    在Linux、macOS和Windows上安装Redis
aliases:
- /docs/getting-started/installation
- /docs/getting-started/tutorial
---

这是一个安装指南。您将学习如何安装、运行和实验Redis服务器进程。

虽然您可以在以下任何一种平台上安装Redis，但您也可以考虑通过创建一个[免费帐户](https://redis.com/try-free?utm_source=redisio&utm_medium=referral&utm_campaign=2023-09-try_free&utm_content=cu-redis_cloud_users)来使用Redis Cloud。

## 安装 Redis

你如何安装Redis取决于你的操作系统和是否想要将其与Redis Stack和Redis UI捆绑安装在一起。请查看下面最适合你需求的指南：

* [从源代码安装Redis](/docs/install/install-redis/install-redis-from-source)
* [在Linux上安装Redis](/docs/install/install-redis/install-redis-on-linux)
* [在macOS上安装Redis](/docs/install/install-redis/install-redis-on-mac-os)
* [在Windows上安装Redis](/docs/install/install-redis/install-redis-on-windows)
* [使用Redis Stack和RedisInsight安装Redis](/docs/install/install-stack/)


## 测试您是否可以使用CLI连接

一旦你启动了Redis，就可以使用`redis-cli`进行连接。

外部程序通过TCP套接字和Redis特定的协议与Redis通信。这个协议通过不同编程语言的Redis客户端库实现。然而，为了使与Redis的交互更简单，Redis提供了一个命令行实用程序，用于向Redis发送指令。该程序被称为**redis-cli**。

要检查Redis是否正常工作的第一件事是使用redis-cli发送一个PING命令：

```
$ redis-cli ping
PONG
```

运行 **redis-cli** 后跟命令名称和其参数将发送此命令给运行在本地主机端口 6379 上的 Redis 实例。您可以更改 `redis-cli` 使用的主机和端口 - 只需尝试 `--help` 选项以查看使用信息。

另一种有趣的运行`redis-cli`的方式是不带参数：程序将以交互模式启动。您可以键入不同的命令并查看它们的回复。

```
$ redis-cli
redis 127.0.0.1:6379> ping
PONG
```

# 保护Redis

Redis的安全性

默认情况下，Redis会绑定到**所有的接口**，并且没有任何身份验证。如果您在一个与外部互联网和攻击者分开的受控环境中使用Redis，那是可以的。然而，如果一个未强化的Redis暴露在互联网上，那将是一个很大的安全问题。如果您不能百分之百确定您的环境已经得到妥善保护，请按照以下步骤来使Redis更加安全：

1. 确保 Redis 使用的端口（默认是 6379，如果在集群模式下还需要 16379，同时 Sentinel 还需要 26379）被防火墙屏蔽，以防止外界可以访问 Redis。
2. 使用配置文件，并设置 `bind` 指令，确保 Redis 只监听您正在使用的网络接口。例如，如果您只在本地访问 Redis，则只需监听回环接口（127.0.0.1），依此类推。
3. 使用 `requirepass` 选项添加额外的安全层，以便客户端需要使用 `AUTH` 命令进行身份验证。
4. 如果您的环境需要加密，请使用 [spiped](http://www.tarsnap.com/spiped.html) 或其他 SSL 隧道软件，以加密 Redis 服务器和 Redis 客户端之间的流量。

请注意，没有任何安全措施的Redis实例[很容易被滥用](http://antirez.com/news/96)，因此请确保你理解上述内容并至少应用一层防火墙。在防火墙就位后，尝试使用`redis-cli`从外部主机连接以证明该实例实际上是无法访问的。

## 从应用程序中使用Redis

当然，仅仅在命令行界面中使用Redis是不够的，因为我们的目标是从应用程序中使用它。为了实现这一目标，您需要下载并安装适用于您所使用编程语言的Redis客户端库。

在此页面中，您会找到各种语言的[完整客户端列表](/clients)。


## Redis 持久化

您可以在[此页面上学习Redis持久化的工作原理](/docs/management/persistence/)，但是对于快速入门来说，重要的是要明白，默认情况下，如果您使用默认配置启动Redis，Redis会间歇性地自动保存数据集（例如，在至少五分钟内进行至少100次数据更改后），因此，如果您希望数据库持久化并在重新启动后重新加载，确保每次想强制进行数据集快照时手动调用**SAVE**命令。否则，请确保使用**SHUTDOWN**命令关闭数据库。

```
$ redis-cli shutdown
```

使用此方式，Redis 将确保在退出之前将数据保存在磁盘上。强烈建议阅读[persistence页面](/docs/management/persistence/)以更好地理解Redis持久化的工作原理。

## 更加规范地安装Redis

直接从命令行运行Redis可以用于一些简单的操作或开发。然而，最终你将在真实服务器上运行一些实际的应用程序。对于这种使用场景，你有两种不同的选择:

* 使用screen运行Redis。
* 在你的Linux系统中使用init脚本以正确的方式安装Redis，这样在重新启动后一切都会正常启动。

强烈推荐使用 init 脚本进行正确安装。

{{% alert title="Note" color="warning" %}}
支持的Linux发行版已经包含了从`/etc/init`启动Redis服务器的功能。
{{% /alert  %}}

{{% alert title="Note" color="warning" %}}
本节剩余部分假设您已从源代码[安装Redis](/docs/install/install-redis-from-source/)。如果您安装的是Redis Stack，则需要下载[一个基本的初始化脚本](https://raw.githubusercontent.com/redis/redis/7.2/utils/redis_init_script)，然后根据Redis Stack在您的平台上的安装方式来调整脚本和以下说明。例如，在Ubuntu 20.04 LTS上，Redis Stack安装在`/opt/redis-stack`，而不是`/usr/local`，因此您需要相应调整。
{{% /alert  %}}

以下说明可以用来使用Redis源代码中附带的init脚本进行正确安装，路径为`/path/to/redis-stable/utils/redis_init_script`。

如果在构建 Redis 源代码后尚未运行 `make install` 命令，您需要在继续之前执行该命令。默认情况下，`make install` 命令会将 `redis-server` 和 `redis-cli` 可执行文件复制到 `/usr/local/bin` 目录中。

* 在其中创建一个目录来存储Redis配置文件和数据：

    ```
    sudo mkdir /etc/redis
    sudo mkdir /var/redis
    ```

* 将您在Redis发行版的**utils**目录下找到的初始化脚本复制到`/etc/init.d`中。我们建议使用运行此Redis实例的端口名称来命名它。确保生成的文件具有`0755`权限。
    
    ```
    sudo cp utils/redis_init_script /etc/init.d/redis_6379
    ```

* 编辑初始化脚本。

    ```
    sudo vi /etc/init.d/redis_6379
    ```

请确保将**REDISPORT**变量设置为您正在使用的端口。
PID文件路径和配置文件名都取决于端口号。

* 将Redis分发根目录中的模板配置文件复制到`/etc/redis/`目录中，以端口号作为名称，例如：

    ```
    sudo cp redis.conf /etc/redis/6379.conf
    ```

* 在`/var/redis`目录下创建一个目录，该目录将作为Redis实例的数据和工作目录：

    ```
    sudo mkdir /var/redis/6379
    ```

* 修改配置文件，确保执行以下更改：
    * 将 **daemonize** 设置为 yes（默认为 no）。
    * 将 **pidfile** 设置为 `/var/run/redis_6379.pid`，根据需要修改端口。
    * 根据需要更改 **port**。在我们的示例中，不需要更改，因为默认端口已经是 `6379`。
    * 设置您首选的 **loglevel**。
    * 将 **logfile** 设置为 `/var/log/redis_6379.log`。
    * 将 **dir** 设置为 `/var/redis/6379`（非常重要的步骤！）
* 最后，使用以下命令将新的 Redis 启动脚本添加到所有默认运行级别：

    ```
    sudo update-rc.d redis_6379 defaults
    ```

你完成了！现在你可以尝试运行你的实例：


```
sudo /etc/init.d/redis_6379 start
```

确保一切正常运行：

1. 在 `redis-cli` 会话中使用 `PING` 命令尝试对实例进行 ping 测试。
2. 使用 `redis-cli save` 进行测试保存，并检查是否正确保存了一个 dump 文件到 `/var/redis/6379/dump.rdb`。
3. 检查你的 Redis 实例是否将日志记录到 `/var/log/redis_6379.log` 文件中。
4. 如果是一台新的机器，你可以在没有问题的情况下尝试它，请确保在重启后仍然正常工作。

{{% alert title="Note" color="warning" %}}
上述说明并没有包括所有可能更改的Redis配置参数。例如，您可以使用AOF持久性而不是RDB持久性，或设置复制等等。
{{% /alert  %}}

您还应该阅读示例 [redis.conf](/docs/management/config-file/) 文件，它有详细的注释，以帮助您进行更改。更多细节也可以在此网站的 [配置文章](/docs/management/config/) 中找到。

示例：

<hr>
