---
title: "Redis哈希"
linkTitle: "哈希"
weight: 40
description: >
    Redis哈希介绍
---

Redis哈希是以字段-值对的形式组织的记录类型。
您可以使用哈希来表示基本对象，并存储计数器的分组等其他内容。


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

在表示*对象*方面，哈希表非常方便，实际上，你可以在哈希表中放置无限数量的字段（除了可用内存之外），因此可以在应用程序中以多种不同的方式使用哈希表。

命令`HSET`设置哈希中的多个字段，而`HGET`检索一个字段。`HMGET`与`HGET`类似，但返回一个值的数组：

{{< clients-example hash_tutorial hmget >}}
> HMGET bike:1 model price no-such-field
1) "Deimos"
2) "4972"
3) (nil)
{{< /clients-example >}}

有一些命令可以对单个字段执行操作，例如 `HINCRBY` ：

{{< clients-example hash_tutorial hincrby >}}
> HINCRBY bike:1 price 100
(integer) 5072
> HINCRBY bike:1 price -100
(integer) 4972
{{< /clients-example >}}

您可以在文档中找到[哈希命令的完整列表](https://redis.io/commands#hash)。

值得注意的是，小哈希（即具有小值的几个元素）在内存中以特殊方式进行编码，使其非常节省存储空间。

## 基本命令

* `HSET` 设置散列中一个或多个字段的值。
* `HGET` 返回给定字段的值。
* `HMGET` 返回一个或多个给定字段的值。
* `HINCRBY` 将给定字段的值增加指定的整数。

查看[哈希命令的完整列表](https://redis.io/commands/?group=hash)。


## 示例

* 存储了自行车：1骑行次数、已发生的事故次数和已更换过的车主数的计数器：
{{< clients-example hash_tutorial incrby_get_mget >}}
> HINCRBY bike:1:stats rides 1
(integer) 1
> HINCRBY bike:1:stats rides 1
(integer) 2
> HINCRBY bike:1:stats rides 1
(integer) 3
> HINCRBY bike:1:stats crashes 1
(integer) 1
> HINCRBY bike:1:stats owners 1
(integer) 1
> HGET bike:1:stats rides
"3"
> HMGET bike:1:stats owners crashes
1) "1"
2) "1"
{{< /clients-example >}}


## 性能

大多数 Redis 哈希命令的时间复杂度为 O(1)。

一些命令 - 如`HKEYS`、`HVALS`和`HGETALL`- 的时间复杂度为O(n)，其中n为字段-值对的数量。

## 限制

每个哈希可以存储最多4,294,967,295（2^32 - 1）个字段-值对。
实际上，您的哈希仅受Redis部署所在虚拟机的总内存限制。

## 了解更多

* [Redis Hashes Explained（Redis 哈希类型讲解）](https://www.youtube.com/watch?v=-KdITaRkQ-U) 是一个简短且全面的视频讲解，涵盖了 Redis 哈希类型。
* [Redis University的RU101（Redis 大学 RU101）](https://university.redis.com/courses/ru101/) 详细介绍 Redis 哈希类型。
