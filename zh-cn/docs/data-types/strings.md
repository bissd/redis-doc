---
title: "Redis 字符串"
linkTitle: "字符串"
weight: 10
description: >
    Redis字符串简介
---

Redis字符串存储字节序列，包括文本、序列化对象和二进制数组。
因此，字符串是您可以与Redis键关联的最简单类型的值。
它们通常用于缓存，但它们还支持额外的功能，可以实现计数器和执行位操作。

由于Redis键是字符串，当我们将字符串类型作为值时，我们将一个字符串映射到另一个字符串。字符串数据类型在许多用例中非常有用，比如缓存HTML片段或页面。

{{< clients-example set_tutorial set_get >}}
    > SET bike:1 Deimos
    OK
    > GET bike:1
    "Deimos"
{{< /clients-example >}}

如您所见，使用`SET`和`GET`命令是我们设置和检索字符串值的方法。请注意，如果键已经存在，即使与非字符串值相关联，`SET`将替换已存在的值。因此，`SET`执行的是赋值操作。

值可以是各种类型的字符串（包括二进制数据），例如您可以将jpeg图像存储在值中。一个值的大小不能超过512 MB。

`SET` 命令有一些有趣的选项，这些选项作为额外的参数提供。例如，我可以要求 `SET` 在键已经存在的情况下失败，或者相反，只有在键已经存在的情况下成功才会执行：

{{< clients-example set_tutorial setnx_xx >}}
    > set bike:1 bike nx
    (nil)
    > set bike:1 bike xx
    OK
{{< /clients-example >}}

有一些其他的字符串操作命令。例如，`GETSET`命令将一个键设置为一个新值，并返回旧值作为结果。例如，如果您有一个系统，每当您的网站接收到一个新访客时，它就会使用`INCR`递增一个Redis键，您可以使用此命令。您可能希望每隔一个小时收集一次这些信息，而不会丢失一个递增。您可以使用`GETSET`命令将键分配为新值"0"并读取旧值。

使用单个命令设置或获取多个键的值，对于减少延迟也非常有用。出于这个原因，有 `MSET` 和 `MGET` 命令。

{{< clients-example set_tutorial mset >}}
    > mset bike:1 "Deimos" bike:2 "Ares" bike:3 "Vanth"
    OK
    > mget bike:1 bike:2 bike:3
    1) "Deimos"
    2) "Ares"
    3) "Vanth"
{{< /clients-example >}}

使用 `MGET` 命令时，Redis 返回一个值的数组。

### 字符串作为计数器
即使字符串是Redis的基本值，你仍然可以执行一些有趣的操作。例如，原子递增是其中之一：

{{< clients-example set_tutorial incr >}}
    > set total_crashes 0
    OK
    > incr total_crashes
    (integer) 1
    > incrby total_crashes 10
    (integer) 11
{{< /clients-example >}}

`INCR`命令将字符串值解析为整数，将其递增一，并将获得的值设置为新值。其他类似的命令包括`INCRBY`，`DECR`以及`DECRBY`。内部上它们始终是相同的命令，只是以稍微不同的方式操作。

什么意思是INCR是原子性的？
即使多个客户端针对相同的键发出INCR命令，也永远不会进入竞争状态。例如，永远不会发生客户1读取"10"，客户2同时读取"10"，两者都将其增加到11，并将新值设置为11的情况。最终值将始终为12，并且在所有其他客户端不执行命令的同时，执行读取-增加-设置操作。


## 限制

默认情况下，Redis字符串的最大长度为512MB。

## 基本命令

### 获取和设置字符串

* `SET` 存储字符串值。
* `SETNX` 仅在键不存在时存储字符串值。用于实现锁定。
* `GET` 获取字符串值。
* `MGET` 一次性检索多个字符串值。

### 管理计数器

* `INCRBY` 原子性地增加（当传递负数时递减）存储在给定键上的计数器。
* 另一个用于浮点计数器的命令是 `INCRBYFLOAT`。

### 位运算

要在字符串上执行位操作，请参阅[位图数据类型](/docs/data-types/bitmaps)文档。

请查看[完整的字符串命令列表](/commands/?group=string)。

## 性能

大多数字符串操作的时间复杂度为 O(1)，这意味着它们非常高效。
但是要小心使用 `SUBSTR`、`GETRANGE` 和 `SETRANGE` 命令，它们的时间复杂度可能为 O(n)。
这些随机访问字符串命令在处理大字符串时可能会引起性能问题。

## 替代方案

如果您将结构化数据存储为序列化字符串，您也可以考虑使用Redis [哈希](/docs/data-types/hashes)或[JSON](/docs/stack/json)。

## 了解更多

* [Redis字符串详解](https://www.youtube.com/watch?v=7CUt4yWeRQE)是一个简短而全面的关于Redis字符串的视频解释者。
* [Redis大学的RU101](https://university.redis.com/courses/ru101/)详细介绍了Redis字符串。
