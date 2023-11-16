---
title: "在Windows上安装Redis"
linkTitle: "Windows"
weight: 1
description: 在Windows上使用Redis进行开发。
aliases:
- /docs/getting-started/installation/install-redis-on-windows/
---

Redis在Windows上没有官方支持。但是，您可以按照下面的说明，在Windows上安装Redis用于开发。

要在Windows上安装Redis，首先需要启用WSL2（Windows子系统Linux）。 WSL2允许您在Windows上本地运行Linux二进制文件。要使用此方法，您需要运行Windows 10版本2004及更高版本或Windows 11。

## 安装或启用WSL2

Microsoft提供了[安装WSL的详细说明](https://docs.microsoft.com/en-us/windows/wsl/install)。按照这些说明操作，并注意它安装的默认Linux发行版。本指南假设使用Ubuntu。

## 安装 Redis

一旦在Windows上运行Ubuntu，可以按照[在Ubuntu/Debian上安装](/docs/install/install-redis-on-linux#install-on-ubuntu-debian)的详细步骤从官方的`packages.redis.io` APT软件库安装Redis的最新稳定版本。
将该软件库添加到<code>apt</code>索引中，更新索引，然后进行安装：

{{< highlight bash  >}}
curl -fsSL https://packages.redis.io/gpg | sudo gpg --dearmor -o /usr/share/keyrings/redis-archive-keyring.gpg

echo "deb [signed-by=/usr/share/keyrings/redis-archive-keyring.gpg] https://packages.redis.io/deb $(lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/redis.list

sudo apt-get update
sudo apt-get install redis
{{< / highlight  >}}

最后，像这样启动Redis服务器：

{{< highlight bash  >}}
sudo service redis-server start
{{< / highlight  >}}

## 连接Redis

您可以通过使用Redis CLI连接来测试您的Redis服务器是否正在运行：

{{< highlight bash  >}}
redis-cli 
127.0.0.1:6379> ping
PONG
{{< / highlight >}}
