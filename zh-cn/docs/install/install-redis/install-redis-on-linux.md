---
title: "在Linux上安装Redis"
linkTitle: "Linux"
weight: 1
description: >
    如何在Linux上安装Redis
aliases:
- /docs/getting-started/installation/install-redis-on-linux
---

大多数主要的Linux发行版都提供Redis的软件包。

## 在Ubuntu/Debian上安装

您可以从官方的 `packages.redis.io` APT 仓库安装最新稳定版本的 Redis。

{{% alert title="Prerequisites" color="warning" %}}
如果你正在运行一个非常精简的发行版（比如 Docker 容器），你可能需要先安装 `lsb-release`、`curl` 和 `gpg`：


{{< highlight bash  >}}
sudo apt install lsb-release curl gpg
{{< / highlight  >}}
{{% /alert  %}}

将存储库添加到 <code>apt</code> 索引中，然后更新并安装：

{{< highlight bash  >}}
curl -fsSL https://packages.redis.io/gpg | sudo gpg --dearmor -o /usr/share/keyrings/redis-archive-keyring.gpg

echo "deb [signed-by=/usr/share/keyrings/redis-archive-keyring.gpg] https://packages.redis.io/deb $(lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/redis.list

sudo apt-get update
sudo apt-get install redis
{{< / highlight  >}}

## 从Snapcraft安装

[Snapcraft商店](https://snapcraft.io/store)提供了可以安装在支持snap的平台上的[Redis包](https://snapcraft.io/redis)。
Snap在大多数主要Linux发行版上都得到支持和可用。

要通过snap进行安装，请运行以下命令：

{{< highlight bash  >}}
sudo snap install redis
{{< / highlight  >}}

如果您的Linux当前没有安装snap，请按照[安装snapd](https://snapcraft.io/docs/installing-snapd)的说明进行安装。
