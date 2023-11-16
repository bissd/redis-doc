---
title: "Redis 和 Gopher 协议"
linkTitle: "Gopher协议"
weight: 10
description: Redis Gopher 协议实现
aliases:
  - /topics/gopher
---

**注意：Redis 7.0 中已移除对 Gopher 的支持**

Redis 包含了对 Gopher 协议的实现，具体规定见 [RFC 1436](https://www.ietf.org/rfc/rfc1436.txt)。

一九九零年代末期，Gopher协议非常流行。它是一种替代Web的协议，而且服务器和客户端的实现非常简单，Redis服务器只需100行代码就能实现这种支持。

如今你对Gopher有什么看法？嗯，Gopher从来没有*真正*消亡，最近有一个运动旨在复兴由纯文本文档组成的更具层次性的Gopher内容。有些人希望拥有一个更简单的互联网，还有些人认为主流互联网过于受控，所以为那些渴望一点新鲜空气的人创造一个替代空间很酷。

无论如何，为了Redis的十岁生日，我们给它作为礼物赠送了Gopher协议。

## 它是如何工作的

Redis Gopher支持使用了Redis的行内协议，并且特别处理了两种非法的行内请求：空请求和以斜杠“/”开头的请求（没有以斜杠开头的Redis命令）。常规的RESP2/RESP3请求完全不受影响，并像往常一样进行处理。

如果在启用了 Gopher 的情况下，您打开到 Redis 的连接并发送一个像 "/foo" 这样的字符串，如果存在一个名为 "/foo" 的键，它将通过 Gopher 协议进行服务。

为了创建一个真正的 Gopher "hole"（Gopher 站点的名字），你可能需要一个脚本，比如 [https://github.com/antirez/gopher2redis](https://github.com/antirez/gopher2redis) 中的脚本。

## 安全警告

如果您计划将Redis放在一个公开可访问的地址上以提供Gopher页面，请确保设置实例的密码。一旦设置了密码：

1. Gopher 服务器（默认情况下已禁用）将通过 Gopher 协议提供内容。
2. 但在客户端进行身份验证之前，无法调用其他命令。

使用`requirepass`选项来保护您的实例。

要启用Gopher支持，请使用以下配置行。

    gopher-enabled yes

访问不是字符串或不存在的键将在Gopher协议格式中生成错误。
