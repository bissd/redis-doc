---
title: "从源码安装Redis"
linkTitle: "源代码"
weight: 5
description: >
    从源代码编译和安装Redis
aliases:
- /docs/getting-started/installation/install-redis-from-source
---

你可以从源代码编译和安装Redis，适用于多种平台和操作系统，包括Linux和macOS。Redis除了C编译器和"libc"库外，没有其他依赖项。

从源文件库中下载文件

Redis源文件可以从[下载页面](/download)获取。您可以通过与[redis-hashes git存储库](https://github.com/redis/redis-hashes)中的摘要进行验证来确认这些下载文件的完整性。

要从Redis下载站点获取最新稳定版本的Redis源文件，请运行以下命令：

{{< highlight bash >}}
wget https://download.redis.io/redis-stable.tar.gz
{{< / highlight >}}

## 编译Redis

要编译Redis，首先下载tarball，然后进入根目录，接着运行`make`命令：

{{< highlight bash >}}
tar -xzvf redis-stable.tar.gz
cd redis-stable
make
{{< / highlight >}}

如果编译成功，你会在"src"目录中找到几个Redis二进制文件，包括：

* **redis-server**：Redis 服务器本身
* **redis-cli** 是与 Redis 进行交互的命令行界面工具。

要将这些二进制文件安装到`/usr/local/bin`，请运行：

{{<highlight bash>}}
sudo make install
{{< /highlight>}}

### 在前台启动和停止 Redis

安装完成后，您可以通过运行以下命令来启动Redis:

{{< highlight bash  >}}
redis-server
{{< / highlight >}}

如果成功，您将会看到Redis的启动日志，并且Redis将在前台运行。

要停止Redis，请输入`Ctrl-C`。

要进行更完整的安装，请继续按照[这些说明](/docs/install/#install-redis-more-properly)操作。
