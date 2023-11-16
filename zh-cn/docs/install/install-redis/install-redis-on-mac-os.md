---
title: "在 macOS 上安装 Redis"
linkTitle: "MacOS"
weight: 1
description: 使用 Homebrew 在 macOS 上安装并启动 Redis。
aliases:
- /docs/getting-started/installation/install-redis-on-mac-os
---

使用Homebrew在macOS上安装Redis的指南。Homebrew是在macOS上安装Redis的最简单方法。如果您更喜欢从源文件在macOS上构建Redis，请参阅[从源安装Redis](/docs/install/install-redis/install-redis-from-source)。

## 先决条件

首先，确保您已经安装了Homebrew。从终端运行：

{{< highlight bash  >}}
brew --version
{{< / highlight >}}

如果此命令失败，您需要按照[Homebrew安装指南](https://brew.sh/)进行操作。

安装方式

在终端中运行：

{{< highlight bash  >}}
brew install redis
{{< / highlight >}}

以下是将Redis安装到您的系统上的步骤。

以前台方式启动和停止Redis

为了测试您的Redis安装情况，您可以在命令行中运行 `redis-server` 可执行文件：

{{< highlight bash  >}}
redis-server
{{< / highlight >}}

如果成功，您将看到Redis的启动日志，并且Redis将在前台运行。

要停止Redis，请输入`Ctrl-C`。

### 使用launchd启动和停止Redis

作为运行 Redis 前台的替代方案，您还可以使用 `launchd` 在后台启动进程：

{{< highlight bash >}}
brew services start redis
{{< /highlight >}}

这将启动 Redis 并在登录时重新启动它。您可以通过运行以下命令来检查由 `launchd` 管理的 Redis 的状态：

{{< highlight bash  >}}
brew services info redis
{{< / highlight >}}

如果服务正在运行，您将看到类似以下的输出：

{{< highlight bash  >}}
redis (homebrew.mxcl.redis)
Running: ✔
Loaded: ✔
User: miranda
PID: 67975
{{< / highlight >}}

要停止服务，请运行：

{{< highlight bash  >}}
brew services stop redis
{{< / highlight >}}

## 连接到 Redis

Redis 运行后，可以通过运行 `redis-cli` 来进行测试：

{{< highlight bash  >}}
redis-cli
{{< / highlight >}}

这将打开 Redis REPL。尝试运行一些命令：

{{< highlight bash >}}
127.0.0.1:6379> lpush demos redis-macOS-demo
OK
127.0.0.1:6379> rpop demos
"redis-macOS-demo"
{{< / highlight >}}

## 下一步操作

一旦您有一个正在运行的Redis实例，您可能会想要：

* 尝试 Redis CLI 教程
* 使用其中一个 Redis 客户端进行连接
