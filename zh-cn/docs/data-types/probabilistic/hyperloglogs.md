---
title: "HyperLogLog"
linkTitle: "HyperLogLog"
weight: 1
description: >
    HyperLogLog是一种概率数据结构，用于估算一个集合的基数。
aliases:
    - /docs/data-types/hyperloglogs/
---

HyperLogLog是一种概率数据结构，可以估计集合的基数。作为一种概率数据结构，HyperLogLog以高效利用空间为代价来进行准确度的折衷。

Redis HyperLogLog 的实现使用高达12 KB，并提供0.81%的标准误差。

计数独特项通常需要相应数量的内存，与您要计数的项数成比例，因为您需要记住您过去已经看到的元素，以避免重复计数它们。然而，存在一组可以以精确度换内存的算法：它们返回带有标准误差的估计度量，就 Redis 实现的HyperLogLog 而言，标准误差小于1%。该算法的神奇之处在于，您不再需要使用与计数项数量成比例的内存量，而是可以使用恒定的内存量；最坏情况下为12k字节，如果您的 HyperLogLog（现在我们简称为逻辑日志）看到的元素很少，那么内存量会少得多。

在Redis中，HLL（HyperLogLog）虽然在技术上是一种不同的数据结构，但它被编码为Redis字符串，所以你可以使用`GET`命令将HLL序列化，并使用`SET`命令将其反序列化到服务器。

概念上，HLL API 就像使用 Set 来执行相同的任务。您可以将每个观察到的元素添加到一个集合中，然后使用 SCARD 来检查集合中的元素数量，这些元素是唯一的，因为 SADD 不会重新添加现有的元素。

您不会真正将*项添加*到HLL中，因为该数据结构仅包含不包括实际元素的状态，但API是相同的：

* 每次看到一个新元素，都可以使用`PFADD`将其添加到计数中。
* 当您想要检索使用`PFADD`命令添加的唯一元素的当前近似值时，可以使用`PFCOUNT`命令。如果您需要合并两个不同的HLLs，则可以使用`PFMERGE`命令。由于HLLs提供唯一元素的近似计数，合并的结果将给出两个源HLL中唯一元素数量的近似值。

{{< clients-example hll_tutorial pfadd >}}
> PFADD bikes Hyperion Deimos Phoebe Quaoar
(integer) 1
> PFCOUNT bikes
(integer) 4
> PFADD commuter_bikes Salacia Mimas Quaoar
(integer) 1
> PFMERGE all_bikes bikes commuter_bikes
OK
> PFCOUNT all_bikes
(integer) 6
{{< /clients-example >}}

以下是一些使用这种数据结构的例子：每天统计用户在搜索表单中执行的唯一查询次数，网页的独立访问者数量以及其他类似情况。

Redis还可以执行HLL的联合操作，请查阅[完整文档](/commands#hyperloglog)获取更多信息。

## 使用案例

**匿名独特访问者网页（SaaS，分析工具）**

这个应用程序回答以下问题：

这一天这个页面有多少独立访问？
这首歌曲有多少独立用户播放？
这个视频有多少独立用户观看？

{{% alert title="Note" color="warning" %}}
 
在一些国家，存储IP地址或任何其他类型的个人识别符是违法的，这使得在您的网站上无法获得独立访客统计数据。

{{% /alert %}}

每个页面（视频/歌曲）每个周期创建一个HyperLogLog，并在每次访问时将每个IP /标识符添加到其中。

## 基本命令

* `PFADD` 将项目添加到 HyperLogLog 中。
* `PFCOUNT` 返回集合中项目数量的估计值。
* `PFMERGE` 将两个或多个 HyperLogLogs 结合成一个。

请查看[HyperLogLog命令的完整列表](https://redis.io/commands/?group=hyperloglog)。

## 性能

写入(`PFADD`)和读取(`PFCOUNT`) HyperLogLog 的时间和空间复杂度都是常数。
合并 HLL 的时间复杂度为 O(n)，其中 _n_ 是草图的数量。

## 限制

HyperLogLog 能够估计具有多达 18,446,744,073,709,551,616（2^64）个成员的集合的基数。

## 了解更多

* [Redis 新数据结构：HyperLogLog](http://antirez.com/news/75) 提供了关于数据结构和其在 Redis 中的实现的详细信息。
* [Redis HyperLogLog 解释](https://www.youtube.com/watch?v=MunL8nnwscQ) 演示了如何使用 Redis HyperLogLog 数据结构构建流量热图。

