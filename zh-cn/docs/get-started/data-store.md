---
title: "Redis作为内存数据结构存储快速入门指南"
linkTitle: "数据结构存储"
weight: 1
description: 了解如何使用基本的Redis数据类型
---

本快速入门指南向您展示如何：

1. 开始使用Redis
2. 在Redis中使用键存储数据
3. 从Redis中使用键检索数据
4. 扫描键空间以查找与特定模式匹配的键

本文的示例是关于一个简单的自行车库存。

# 设置

开始使用Redis最简单的方法是使用Redis Cloud：

1. 创建一个[免费账户](https://redis.com/try-free?utm_source=redisio&utm_medium=referral&utm_campaign=2023-09-try_free&utm_content=cu-redis_cloud_users)。
2. 按照说明创建一个免费数据库。
   
![展示1](../img/free-cloud-db.png)

您还可以按照[安装指南](/docs/install/install-stack/)在本地机器上安装Redis。

## 连接

第一步是连接到Redis。您可以在本文档站点的[连接部分](/docs/connect)中找到有关连接选项的更多详细信息。以下示例显示如何连接到在本地主机上运行（`-h 127.0.0.1`）并监听默认端口（`-p 6379`）的Redis服务器：

{{< clients-example search_quickstart connect >}}
> redis-cli -h 127.0.0.1 -p 6379
{{< /clients-example>}}
<br/>
{{% alert title="Tip" color="warning" %}}
您可以从Redis Cloud数据库配置页面复制并粘贴连接详细信息。以下是一个Cloud数据库的连接字符串示例，该数据库托管在AWS地区`us-east-1`，并监听端口16379：`redis-16379.c283.us-east-1-4.ec2.cloud.redislabs.com:16379`。连接字符串的格式为`host:port`。您还必须复制并粘贴Cloud数据库的用户名和密码，然后将凭证传递给您的客户端，或者在连接建立后使用[AUTH命令](/commands/auth/)。
{{% /alert  %}}

## 存储和检索数据

# Redis 远程字典服务器

- Redis 是远程字典服务器的缩写。
- 在 Redis 中，你可以使用与本地编程环境相同的数据类型，但是在服务器端运行。

与字节数组类似，Redis字符串存储字节序列，包括文本、序列化对象、计数器值和二进制数组。以下示例向您展示了如何设置和获取字符串值：

{{< clients-example set_and_get >}}
SET bike:1 "Process 134"
GET bike:1
{{< /clients-example >}}

哈希是字典（dict或哈希映射）的等价物。除其他用途外，你可以使用哈希来表示普通对象并存储计数器的分组。下面的示例解释了如何设置和访问对象的字段值：

{{< clients-example hash_tutorial set_get_all >}}
> HSET bike:1 model Deimos brand Ergonom type 'Enduro bikes' price 4972
(integer) 4
> HGET bike:1 model
"Deimos"
> HGET bike:1 price
"4972"
> HGETALL bike:1
1) "model"
2) "Deimos"
3) "brand"
4) "Ergonom"
5) "type"
6) "Enduro bikes"
7) "price"
8) "4972"
{{< /clients-example >}}

您可以在本文档网站的[数据类型部分](/docs/data-types/)中获得可用数据类型的完整概述。每种数据类型都有命令，允许您操作或检索数据。[命令参考](/commands/)提供了详细的解释。

## 扫描键空间

每个 Redis 中的项目都有一个唯一的键。所有项目都存在于 Redis [键空间](/docs/manual/keyspace/) 中。您可以通过 [SCAN 命令](/commands/scan/) 扫描 Redis 的键空间。以下是一个示例，扫描具有前缀 `bike:` 的前 100 个键：

{{< clients-example scan_example >}}
SCAN 0 MATCH "bike:*" COUNT 100
{{< /clients-example >}}

[SCAN](/commands/scan/) 返回一个光标位置，允许您进行迭代扫描，直到达到光标值 0 的下一批键。

## 下一步

通过了解Redis Stack，您可以应用于更多的使用情况。以下是两个其他的快速入门指南：



* [Redis作为文档数据库](/docs/get-started/document-database/)
* [Redis作为向量数据库](/docs/get-started/vector-database/)
