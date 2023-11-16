---
title: "Redis集合"
linkTitle: "集合"
weight: 30
description: >
    Redis集合介绍
---

Redis集合是一个无序的独特字符串的集合。
您可以使用Redis集合来高效地进行以下操作:

* 跟踪唯一的项目（例如，跟踪访问给定博客文章的所有唯一IP地址）。
* 表示关系（例如，具有给定角色的所有用户的集合）。
* 执行常见的集合操作，如交集、并集和差集。

## 基础命令

* `SADD` 添加一个新成员到集合中。
* `SREM` 从集合中移除指定的成员。
* `SISMEMBER` 测试一个字符串是否是集合的成员。
* `SINTER` 返回两个或多个集合共有的成员（即交集）。
* `SCARD` 返回集合的大小（也称为基数）。

查看[完整的集合命令列表](https://redis.io/commands/?group=set)。

## 示例

* 存储在法国和美国的自行车比赛集合。注意，如果添加一个已经存在的成员，它将被忽略。
{{< clients-example sets_tutorial sadd >}}
> SADD bikes:racing:france bike:1
(integer) 1
> SADD bikes:racing:france bike:1
(integer) 0
> SADD bikes:racing:france bike:2 bike:3
(integer) 2
> SADD bikes:racing:usa bike:1 bike:4
(integer) 2
{{< /clients-example >}}

* 检查自行车1或自行车2是否在美国参加比赛。
{{< clients-example sets_tutorial sismember >}}
> SISMEMBER bikes:racing:usa bike:1
(integer) 1
> SISMEMBER bikes:racing:usa bike:2
(integer) 0
{{< /clients-example >}}

* 哪些自行车参加了这两场比赛？
{{< clients-example sets_tutorial sinter >}}
> SINTER bikes:racing:france bikes:racing:usa
1) "bike:1"
{{< /clients-example >}}

* 在法国有多少辆自行车比赛？
{{< clients-example sets_tutorial scard >}}
> SCARD bikes:racing:france
(integer) 3
{{< /clients-example >}}

## 教程

`SADD`命令向集合中添加新元素。还可以对集合执行许多其他操作，如测试给定元素是否已经存在，执行多个集合之间的交集、并集或差集等操作。

{{< clients-example sets_tutorial sadd_smembers >}}
> SADD bikes:racing:france bike:1 bike:2 bike:3
(integer) 3
> SMEMBERS bikes:racing:france
1) bike:3
2) bike:1
3) bike:2
{{< /clients-example >}}

我在集合中添加了三个元素，并告诉Redis返回所有元素。集合没有顺序保证，Redis可以在每次调用时以任何顺序返回元素。

Redis有用于测试集合成员的命令。这些命令可以用于单个项目和多个项目：

{{< clients-example sets_tutorial smismember >}}
> SISMEMBER bikes:racing:france bike:1
(integer) 1
> SMISMEMBER bikes:racing:france bike:2 bike:3 bike:4
1) (integer) 1
2) (integer) 1
3) (integer) 0
{{< /clients-example >}}

我们也可以找出两个集合之间的不同之处。例如，我们可能想知道在法国比赛而不在美国比赛的自行车有哪些：

{{< clients-example sets_tutorial sdiff >}}
> SADD bikes:racing:usa bike:1 bike:4
(integer) 2
> SDIFF bikes:racing:france bikes:racing:usa
1) "bike:3"
2) "bike:2"
{{< /clients-example >}}

有其他一些非常简单实现的非平凡操作，
只需使用正确的 Redis 命令即可。例如，我们可能想要一个包含所有在法国、美国和其他一些赛事中参赛自行车的列表。
我们可以使用 `SINTER` 命令来实现这一点，它可以计算不同集合之间的交集。除了交集，您还可以计算并集、差集等等。
例如，如果我们添加了第三场比赛，我们就可以看到其中一些命令的效果：

{{< clients-example sets_tutorial multisets >}}
> SADD bikes:racing:france bike:1 bike:2 bike:3
(integer) 3
> SADD bikes:racing:usa bike:1 bike:4
(integer) 2
> SADD bikes:racing:italy bike:1 bike:2 bike:3 bike:4
(integer) 4
> SINTER bikes:racing:france bikes:racing:usa bikes:racing:italy
1) "bike:1"
> SUNION bikes:racing:france bikes:racing:usa bikes:racing:italy
1) "bike:2"
2) "bike:1"
3) "bike:4"
4) "bike:3"
> SDIFF bikes:racing:france bikes:racing:usa bikes:racing:italy
(empty array)
> SDIFF bikes:racing:france bikes:racing:usa
1) "bike:3"
2) "bike:2"
> SDIFF bikes:racing:usa bikes:racing:france
1) "bike:4"
{{< /clients-example >}}

您会注意到，当所有集合之间的差集为空时，`SDIFF`命令会返回一个空数组。您还会注意到，传递给`SDIFF`的集合的顺序很重要，因为差集是非交换的。

当你想从一个集合中移除元素时，可以使用 `SREM` 命令从集合中移除一个或多个元素，或者使用 `SPOP` 命令从集合中移除一个随机元素。你也可以使用 `SRANDMEMBER` 命令返回集合中的一个随机元素，而不移除它。

{{< clients-example sets_tutorial srem >}}
> SADD bikes:racing:france bike:1 bike:2 bike:3 bike:4 bike:5
(integer) 3
> SREM bikes:racing:france bike:1
(integer) 1
> SPOP bikes:racing:france
"bike:3"
> SMEMBERS bikes:racing:france
1) "bike:2"
2) "bike:4"
3) "bike:5"
> SRANDMEMBER bikes:racing:france
"bike:2"
{{< /clients-example >}}

## 限制

Redis集合的最大大小是2^32-1（4,294,967,295）个成员。

## 性能

大多数集合操作（包括添加、删除和检查项目是否为集合成员）都是O(1)的。
这意味着它们非常高效。
然而，对于具有数十万个成员或更多的大型集合，运行`SMEMBERS`命令时应谨慎。
此命令为O(n)，并返回单个响应中的整个集合。
作为替代方案，考虑使用`SSCAN`，它可以使您以迭代方式检索集合的所有成员。

## 替代方案

设置大型数据集（或流数据）的成员身份检查可能会使用大量内存。
如果您担心内存使用并且不需要完美的精确度，请考虑使用[布隆过滤器或布谷鸟过滤器](/docs/stack/bloom)替代集合。

Redis集合经常被用作一种索引。
如果你需要索引和查询你的数据，请考虑使用[JSON](/docs/stack/json)数据类型和[搜索和查询](/docs/stack/search)功能。

## 了解更多信息

* [Redis集合的解释](https://www.youtube.com/watch?v=PKdCppSNTGQ)和[Redis集合的详细解释](https://www.youtube.com/watch?v=aRw5ME_5kMY)是两个简短但详细的视频解说，涵盖了Redis集合的内容。
* [Redis大学的RU101课程](https://university.redis.com/courses/ru101/)详细探讨了Redis集合。
