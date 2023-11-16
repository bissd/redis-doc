---
title: "了解 Redis 数据类型"
linkTitle: "了解数据类型"
description: Redis支持的数据类型概述
weight: 35
aliases:
    - /docs/manual/data-types
    - /topics/data-types
    - /docs/data-types/tutorial
---

Redis是一个数据结构服务器。
在其核心，Redis提供了一系列本地数据类型，帮助您解决各种问题，从缓存到队列再到事件处理。
以下是每种数据类型的简短描述，附有更广泛的概述和命令参考链接。

如果您想尝试每种数据结构的全面教程，请查看下面的概览页面。


## 核心

## 字符串

[Redis strings](/docs/data-types/strings) 是最基本的 Redis 数据类型，表示一个字节序列。
了解更多信息，请参见：

* [Redis字符串概述](/docs/data-types/strings/)
* [Redis字符串命令参考](/commands/?group=string)

# 列表

[Redis列表](/docs/data-types/lists)是按插入顺序排序的字符串列表。
详细信息请参考：

* [Redis列表概述](/docs/data-types/lists/)
* [Redis列表命令参考](/commands/?group=list)

### 集合

[Redis 集合](/docs/data-types/sets) 是无序的唯一字符串集合，类似于你喜欢的编程语言中的集合（例如，[Java HashSets](https://docs.oracle.com/javase/7/docs/api/java/util/HashSet.html)，[Python sets](https://docs.python.org/3.10/library/stdtypes.html#set-types-set-frozenset)，等等）。
使用 Redis 集合，你可以在 O(1) 时间内进行添加、删除和存在性测试（换句话说，无论集合元素的数量如何）。
有关更多信息，请参阅：

* [Redis 集合概述](/docs/data-types/sets/)
* [Redis 集合命令参考](/commands/?group=set)

### 哈希

[Redis哈希](/docs/data-types/hashes)是一种由字段-值对组成的记录类型。
因此，Redis哈希类似于[Python字典](https://docs.python.org/3/tutorial/datastructures.html#dictionaries)、[Java HashMaps](https://docs.oracle.com/javase/8/docs/api/java/util/HashMap.html)和[Ruby哈希](https://ruby-doc.org/core-3.1.2/Hash.html)。
欲了解更多信息，请参见：

* [Redis哈希的概述](/docs/data-types/hashes/)
* [Redis哈希命令参考](/commands/?group=hash)

### 有序集合

[Redis 分类集合](/docs/data-types/sorted-sets) 是一组唯一字符串集合，通过每个字符串的关联分数来维持顺序。
了解更多信息，请参阅：

* [Redis有序集合概述](/docs/data-types/sorted-sets)
* [Redis有序集合命令参考](/commands/?group=sorted-set)

### 流

一个[Redis流](/docs/data-types/streams)是一个行为类似于只追加日志的数据结构。
流帮助按发生顺序记录事件，然后进行处理。
更多信息请参阅：

* [Redis Streams 概述](/docs/data-types/streams)
* [Redis Streams 命令参考](/commands/?group=stream)

### 地理空间索引

[Redis地理空间索引](/docs/data-types/geospatial)可用于在给定的地理半径或边界框内查找位置。
更多信息请参见：

* [Redis地理空间索引概述](/docs/data-types/geospatial/)
* [Redis地理空间索引命令参考](/commands/?group=geo)

### 位图

[Redis 位图](/docs/data-types/bitmaps/) 允许您对字符串执行位运算。
更多信息，请参阅：

* [Redis位图的概述](/docs/data-types/bitmaps/)
* [Redis位图命令参考](/commands/?group=bitmap)

### 位字段

[Redis 位域](/docs/data-types/bitfields/) 可以高效地将多个计数器编码为字符串值。
位域提供原子化的获取、设置和递增操作，并支持不同的溢出策略。
更多信息，请参见：

* [Redis bitfields 概览](/docs/data-types/bitfields/)
* `BITFIELD` 命令。

### HyperLogLog
超级日志日志

[Redis HyperLogLog](/docs/data-types/hyperloglogs)数据结构提供了对大型集合的基数（即元素数量）的概率估计。有关更多信息，请参阅：

* [Redis HyperLogLog概述](/docs/data-types/hyperloglogs)
* [Redis HyperLogLog命令参考](/commands/?group=hyperloglog)

## 扩展插件

为了扩展所包含数据类型提供的功能，可以使用以下选项之一：

1. 使用Lua编写自己的自定义[服务器端函数](/docs/manual/programmability/)。
2. 使用[模块API](/docs/reference/modules/)编写自己的Redis模块，或查看[社区支持的模块](/docs/modules/)。
3. 使用[JSON](/docs/stack/json/)、[查询功能](/docs/stack/search/)、[时间序列](/docs/stack/timeseries/)以及其他由[Redis Stack](/docs/stack/)提供的功能。

<hr>
