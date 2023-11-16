---
title: "Redis安全"
linkTitle: "安全"
weight: 1
description: Redis中的安全模型和特性
aliases: [
    /topics/security,
    /docs/manual/security,
    /docs/manual/security.md
]
---

本文档从Redis的角度提供了安全主题的介绍。它涵盖了Redis提供的访问控制、代码安全问题、可以通过选择恶意输入从外部触发的攻击以及其他类似主题。您可以通过参加[Redis University安全课程](https://university.redis.com/courses/ru330/)了解更多关于访问控制、数据保护与加密、安全的Redis架构以及安全的部署技术。

对于与安全相关的联系人，请在GitHub上打开一个问题，或者当您觉得保护通信安全非常重要时，使用本文档末尾的GPG密钥。

## 安全模型

Redis被设计为仅供可信任的客户端在可信任的环境中访问。
这意味着通常情况下，直接将Redis实例暴露在互联网上或一般情况下，
在非可信任的客户端可以直接访问Redis的TCP端口或UNIX套接字的环境中是不明智的。

例如，在使用Redis作为数据库、缓存或消息系统实现的Web应用程序的常见情况下，
应用程序前端（Web端）内的客户端将查询Redis以生成页面或执行由Web应用程序用户请求或触发的操作。

在这种情况下，Web应用程序在Redis和不受信任的客户端之间进行访问中介（用户浏览器访问Web应用程序）。

在一般情况下，不可信的访问Redis应该由实现了ACL的中间层进行调控，验证用户输入，并决定针对Redis实例执行哪些操作。

## 网络安全

对Redis端口的访问应该被拒绝除了网络中的可信客户端之外的其他人，因此运行Redis的服务器应该只能由使用Redis的应用程序所在的计算机直接访问。

在单个计算机直接暴露在互联网上的普通情况下，比如虚拟化的Linux实例（Linode，EC2，...），应该对Redis端口进行防火墙设置，以防止外部访问。客户端仍然可以通过回环接口访问Redis。

请注意，可以通过在**redis.conf**文件中添加以下行来将Redis绑定到单个接口：

    bind 127.0.0.1

无法保护Redis端口免受外部攻击者的攻击可能会造成很大的安全影响，因为Redis的特性。例如，一个外部攻击者可以使用单个`FLUSHALL`命令来删除整个数据集。

## 保护模式

很不幸，许多用户未能保护Redis实例免受外部网络访问。许多实例只是暴露在带有公共IP的互联网上。自从3.2.0版起，当使用默认配置（绑定所有接口）并且没有密码来访问Redis时，Redis进入一种特殊模式，称为**保护模式**。在这种模式下，Redis只回应来自回环接口的查询，并向从其他地址连接的客户端回复一个解释问题及如何正确配置Redis的错误信息。

我们期望保护模式可以严重减少由未经适当管理的Redis实例执行而引起的安全问题。然而，系统管理员仍然可以忽略Redis给出的错误，禁用保护模式或手动绑定所有接口。

# 身份验证

Redis提供了两种验证客户端的方法。
推荐的身份验证方法是在Redis 6中引入的访问控制列表(Access Control Lists)，允许创建命名用户并分配细粒度权限。
在[这里](/docs/management/security/acl/)阅读更多关于访问控制列表的信息。

通过编辑**redis.conf**文件启用传统身份验证方法，并使用`requirepass`设置来提供数据库密码。
然后，所有客户端都将使用该密码。

当启用`requirepass`设置时，Redis将拒绝任何未经身份验证的客户端的查询。客户端可以通过发送密码后跟的**AUTH**命令来进行身份验证。

密码是由系统管理员以明文形式在redis.conf文件中设置的。它应该足够长，以防止暴力破解攻击，原因有两个：

* Redis在处理查询时非常快速。一个外部客户端可以测试每秒测试多个密码。
* Redis密码存储在**redis.conf**文件和客户端配置中。由于系统管理员不需要记住它，密码可以非常长。

验证层的目标是可选择地提供一层冗余。如果防火墙或其他系统用于保护Redis免受外部攻击者的攻击失败，而无需了解验证密码，外部客户端仍将无法访问Redis实例。

由于`AUTH`命令和其他Redis命令一样，都是以未加密形式发送的，所以它不能防止具有足够网络访问权限的攻击者进行窃听。

## TLS 支持

Redis 对所有通信渠道，包括客户端连接、复制链接和 Redis 集群总线协议，都提供了可选的 TLS 支持。

## 禁止特定命令

Redis中可以禁止使用命令或者将命令更名为不可预测的名称，以便常规客户端仅限于指定的一组命令。

例如，虚拟化服务器提供商可以提供托管的Redis实例服务。在这种情况下，普通用户可能不应该能够调用Redis **CONFIG**命令来修改实例的配置，但是提供和删除实例的系统应该可以这样做。

在这种情况下，可以重命名或完全屏蔽命令来自命令表。该功能作为一个语句可用于redis.conf配置文件中。例如：

    rename-command CONFIG b840fc02d524045429941cc15f59e41cb7be6c52

在上面的例子中，**CONFIG**命令的名称被更改为难以猜测的名称。也可以通过将其名称更改为空字符串来完全禁用它（或任何其他命令），如下例所示：

    rename-command CONFIG ""

由外部客户端输入引发的攻击

有一类攻击可以由攻击者从外部触发，即使没有对实例进行外部访问。例如，攻击者可能会向Redis中插入数据，从而触发Redis内部实现的数据结构的病态（最坏情况）算法复杂度。

攻击者可以通过网络表单提供一组已知哈希到同一个哈希桶的字符串，从而将O(1)的预期时间（平均时间）变为O(N)的最坏情况。这可能消耗比预期更多的CPU，最终导致拒绝服务。

为了防止这种特定的攻击，Redis使用每次执行的伪随机种子来哈希函数。

Redis使用qsort算法实现SORT命令。目前，该算法不是随机的，因此通过精心选择正确的输入，可能会触发二次最坏情况行为。

## 字符串转义和NoSQL注入

Redis协议没有字符串转义的概念，因此在正常情况下使用普通的客户端库是不可能发生注入的。
该协议使用带前缀长度的字符串，并且完全支持二进制安全。

由于通过`EVAL`和`EVALSHA`命令执行的Lua脚本遵循相同的规则，因此这些命令也是安全的。

虽然这可能是一个奇怪的用例，但应用程序应避免使用从不受信任的来源获取的字符串来组成Lua脚本的正文。

## 代码安全

在传统的Redis设置中，客户端被允许完全访问命令集，但访问实例不应导致能够控制Redis运行的系统。

Redis在内部使用了所有众所周知的安全编码实践，以防止缓冲区溢出、格式错误和其他内存损坏问题。然而，使用**CONFIG**命令控制服务器配置的能力允许客户端更改程序的工作目录和转储文件的名称。这允许客户端将RDB Redis文件写入随机路径。这是一个[安全问题](http://antirez.com/news/96)，可能导致能够破坏系统和/或以与Redis运行的相同用户身份运行不受信任的代码的能力。

Redis不需要root权限来运行。建议将其作为一个非特权的*redis*用户运行，仅用于此目的。

## GPG密钥

```
-----BEGIN PGP PUBLIC KEY BLOCK-----

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
=J9BD
-----END PGP PUBLIC KEY BLOCK-----
```
