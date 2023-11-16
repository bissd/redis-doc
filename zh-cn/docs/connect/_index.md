---
title: 连接到Redis
linkTitle: 连接
description: 学习如何使用用户界面和客户端库
weight: 35
aliases:
  - /docs/ui
---

您可以通过以下方式连接到Redis：

* 使用`redis-cli`命令行工具
* 使用RedisInsight作为图形用户界面
* 通过编程语言的客户端库
  
## Redis命令行界面

[RDS 命令行接口](/docs/connect/cli)（也被称为 `redis-cli`）是一个终端程序，用于向 Redis 服务器发送命令并读取回复。它具有如下两种主要模式：

1. 用户输入Redis命令并接收回复的交互式Read Eval Print Loop (REPL)模式。
2. 通过执行`redis-cli`命令并附加参数的命令模式，将回复打印到标准输出。

## RedisInsight

[RedisInsight](/docs/connect/insight)结合了图形用户界面和Redis CLI，让您可以与任何Redis部署一起使用。您可以可视化浏览和与数据交互，利用诊断工具，通过示例学习等等。最重要的是，RedisInsight是免费的。

## 客户端库

连接Redis数据库非常简单。官方客户端库支持以下语言：

* [C#/.NET](/docs/connect/clients/dotnet)
* [Go](/docs/connect/clients/go)
* [Java](/docs/connect/clients/java)
* [Node.js](/docs/connect/clients/nodejs)
* [Python](/docs/connect/clients/python)

您可以在[客户端页面](/resources/clients/)上找到包括社区维护的所有客户端库的完整列表。

<hr/>
