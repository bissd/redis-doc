---
title: "Redis Bitfields"
linkTitle: "Bitfields"
weight: 130
description: >
    Redis bitfields 简介
---

Redis位字段允许您设置、递增和获取任意位长度的整数值。
例如，您可以对从无符号1位整数到有符号63位整数的任何值进行操作。

这些值是使用二进制编码的Redis字符串存储的。
位字段支持原子读取、写入和增量操作，使其成为管理计数器和类似数值的良好选择。


## 基本命令

* `BITFIELD` 原子设置、递增和读取一个或多个值。
* `BITFIELD_RO` 是 `BITFIELD` 的只读变体。


## 示例

假设你正在追踪在线游戏中的活动。
你想要为每个玩家保持两个关键指标：总金币数量和杀怪数。
由于你的游戏非常令人上瘾，这些计数器应该至少为32位宽度。

您可以使用每个玩家的一个位字段来表示这些计数器。

* 新玩家从教程开始时拥有1000金币（偏移量为0的计数器）。
```
> BITFIELD player:1:stats SET u32 #0 1000
1) (整数) 0
```

* 在杀死囚禁王子的地精后，将获得的50金币加上并增加“已杀敌人”计数器（偏移量为1）。
```
> BITFIELD player:1:stats INCRBY u32 #0 50 INCRBY u32 #1 1
1) (整数) 1050
2) (整数) 1
```

* 给铁匠支付999金币，购买一把传奇锈剑。
```
> BITFIELD 玩家：1：属性 INCRBY u32 #0 -999
1)（整数）51
```

* 读取玩家的状态:
```
> BITFIELD player:1:stats GET u32 #0 GET u32 #1
1) (integer) 51
2) (integer) 1
```


## 性能

`BITFIELD`的时间复杂度是O(n)，其中_n_是访问的计数器数目。
