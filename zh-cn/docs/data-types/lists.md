---
title: "Redis列表"
linkTitle: "列表"
weight: 20
description: >
    Redis列表简介
---

Redis列表是字符串值的链表。
Redis列表通常用于：

* 实现堆栈和队列。
* 为后台工作系统构建队列管理。

## 基本命令

* `LPUSH` 在列表的头部添加一个新元素； `RPUSH` 在尾部添加。
* `LPOP` 从列表的头部移除并返回一个元素； `RPOP` 从列表的尾部移除并返回一个元素。
* `LLEN` 返回列表的长度。
* `LMOVE` 原子性地将元素从一个列表移动到另一个列表。
* `LTRIM` 缩减列表为指定范围的元素。

### 阻止命令

列表支持几个块命令。
例如：

* `BLPOP` 从列表头部移除并返回一个元素。
  如果列表为空，命令将阻塞直到有元素可用或达到指定的超时时间。
* `BLMOVE` 原子地将元素从源列表移动到目标列表。
  如果源列表为空，命令将阻塞直到有新的元素可用。

查看[完整的列表命令系列](https://redis.io/commands/?group=list)。

## 示例

* 将列表视为队列（先进先出）：
{{< clients-example list_tutorial queue >}}
> LPUSH bikes:repairs bike:1
(integer) 1
> LPUSH bikes:repairs bike:2
(integer) 2
> RPOP bikes:repairs
"bike:1"
> RPOP bikes:repairs
"bike:2"
{{< /clients-example >}}

* 使用列表作为堆栈（先进后出）：
{{< clients-example list_tutorial stack >}}
> LPUSH bikes:repairs bike:1
(integer) 1
> LPUSH bikes:repairs bike:2
(integer) 2
> LPOP bikes:repairs
"bike:2"
> LPOP bikes:repairs
"bike:1"
{{< /clients-example >}}

* 检查列表的长度：
{{< clients-example list_tutorial llen >}}
>LLEN bikes:repairs
(integer) 0
{{< /clients-example >}}

* 从一个列表中原子性地弹出一个元素并推到另一个列表：
{{< clients-example list_tutorial lmove_lrange >}}
> LPUSH bikes:repairs bike:1
(integer) 1
> LPUSH bikes:repairs bike:2
(integer) 2
> LMOVE bikes:repairs bikes:finished LEFT LEFT
"bike:2"
> LRANGE bikes:repairs 0 -1
1) "bike:1"
> LRANGE bikes:finished 0 -1
1) "bike:2"
{{< /clients-example >}}

* 要限制列表的长度，可以调用 `LTRIM`：
{{< clients-example list_tutorial ltrim.1 >}}
> RPUSH bikes:repairs bike:1 bike:2 bike:3 bike:4 bike:5
(integer) 5
> LTRIM bikes:repairs 0 2
OK
> LRANGE bikes:repairs 0 -1
1) "bike:1"
2) "bike:2"
3) "bike:3"
{{< /clients-example >}}

### 什么是列表？

列表是一种用于存储多个有序项的数据结构。列表可以包含任意类型的元素，例如整数、字符串、布尔值等。在列表中，每个元素都有一个唯一的索引，可以通过索引访问特定的元素。使用列表可以方便地处理多个相关的数据，如存储学生的成绩、记录购物清单等。
为了解释列表数据类型，最好从一点理论开始，因为信息技术人员经常以不正确的方式使用术语“列表”。例如，“Python列表”并不是名字所暗示的那样（链表），而是数组（实际上在Ruby中，相同的数据类型被称为数组）。

从非常一般的角度来看，列表只是一系列有序的元素：10,20,1,2,3是一个列表。但是使用数组实现的列表的属性与使用链表实现的列表的属性非常不同。

Redis列表是通过链表实现的。这意味着，即使列表中有数百万个元素，添加新元素到列表头部或尾部的操作也是*常数时间*。使用`LPUSH`命令将新元素添加到包含十个元素的列表头部的速度与将元素添加到包含1000万个元素的列表头部的速度相同。

什么是缺点？使用数组实现的列表通过**索引**访问元素非常快（具有常数时间的索引访问），而使用链表实现的列表则不太快（该操作所需的工作量与访问元素的索引成正比）。

Redis Lists使用链表来实现，因为对于数据库系统来说，在非常快的方式下能够往一个非常长的列表中添加元素是至关重要的。另一个强大的优势是，正如你马上会看到的，Redis Lists可以以恒定时间和恒定长度获取。

当快速访问大型元素集合的中间部分很重要时，可以使用一种称为有序集合的不同数据结构。有序集合在[有序集合](/docs/data-types/sorted-sets)教程页面中有介绍。

### 使用Redis List的第一步

`LPUSH`命令向列表的左侧（头部）添加一个新元素，而`RPUSH`命令向列表的右侧（尾部）添加一个新元素。最后，`LRANGE`命令从列表中提取元素的范围。

{{< clients-example list_tutorial lpush_rpush >}}
> RPUSH bikes:repairs bike:1
(integer) 1
> RPUSH bikes:repairs bike:2
(integer) 2
> LPUSH bikes:repairs bike:important_bike
(integer) 3
> LRANGE bikes:repairs 0 -1
1) "bike:important_bike"
2) "bike:1"
3) "bike:2"
{{< /clients-example >}}

请注意，`LRANGE`命令接受两个索引，用于指定返回的范围，分别是范围的第一个元素和最后一个元素。两个索引都可以为负数，告诉Redis从末尾开始计数：所以-1表示最后一个元素，-2表示倒数第二个元素，以此类推。

正如你所看到的，`RPUSH` 在列表的右侧添加了元素，而最后的 `LPUSH` 在左侧添加了元素。

两个命令都是*可变命令*，意味着你可以在一次调用中自由地将多个元素推入列表中：

{{< clients-example list_tutorial variadic >}}
> RPUSH bikes:repairs bike:1 bike:2 bike:3
(integer) 3
> LPUSH bikes:repairs bike:important_bike bike:very_important_bike
> LRANGE mylist 0 -1
1) "bike:very_important_bike"
2) "bike:important_bike"
3) "bike:1"
4) "bike:2"
5) "bike:3"
{{< /clients-example >}}

Redis 列表上定义的一个重要操作是能够 *弹出元素*。
弹出元素是从列表中检索元素并同时从列表中删除它的操作。
您可以从列表的左侧和右侧弹出元素，类似于您可以在列表的两侧推送元素。
我们将添加三个元素并弹出三个元素，因此在这些命令序列的末尾，列表为空，并且没有更多的元素可供弹出：

{{< clients-example list_tutorial lpop_rpop >}}
> RPUSH bikes:repairs bike:1 bike:2 bike:3
(integer) 3
> RPOP bikes:repairs
"bike:3"
> LPOP bikes:repairs
"bike:1"
> RPOP bikes:repairs
"bike:2"
> RPOP bikes:repairs
(nil)
{{< /clients-example >}}

Redis 返回了一个空值，表示列表中没有元素。

### 列表的常见用例

列表在许多任务中都很有用，下面是两个非常典型的使用场景:

以下是用户发布到社交网络的最新更新。
使用生产者-消费者模式进行进程之间的通信，生产者将项目推送到列表中，消费者（通常是*worker*）消耗这些项目并执行操作。 Redis具有特殊的列表命令，使此用例更可靠和高效。

例如，流行的 Ruby 库 [resque](https://github.com/resque/resque) 和 [sidekiq](https://github.com/mperham/sidekiq) 都使用 Redis 列表来实现后台作业。

这个流行的 Twitter 社交网络将用户发布的最新推文以 Redis 列表的方式接收。

为了逐步描述一个常见的使用案例，请想象一下您的主页显示了一个照片分享社交网络发布的最新照片，并且您想要加快访问速度。

每当用户发布一张新照片，我们使用`LPUSH`将其ID添加到一个列表中。
当用户访问主页时，我们使用`LRANGE 0 9`来获取最新发布的10个项目。

### 截断列表

在许多应用场景中，我们只想使用列表来存储*最新的项目*，无论它们是什么：社交网络更新、日志或其他任何东西。

Redis 允许我们将列表作为有容量限制的集合，只记住最新的 N 个元素，并使用 `LTRIM` 命令丢弃所有最旧的元素。

`LTRIM`命令类似于`LRANGE`，但是**它不会显示指定范围内的元素**，而是将该范围设置为新的列表值。给定范围之外的所有元素都将被删除。

例如，如果您正在在维修列表的末尾添加自行车，但只想关注列表上存在最长时间的3辆自行车：

{{< clients-example list_tutorial ltrim >}}
> RPUSH bikes:repairs bike:1 bike:2 bike:3 bike:4 bike:5
(integer) 5
> LTRIM bikes:repairs 0 2
OK
> LRANGE bikes:repairs 0 -1
1) "bike:1"
2) "bike:2"
3) "bike:3"
{{< /clients-example >}}

上述的`LTRIM`命令告诉Redis只保留列表从索引0到2的元素，其他的将被丢弃。这允许一个非常简单但有用的模式：将列表的推送操作+列表的修剪操作一起使用，添加一个新元素并丢弃超出限制的元素。然后可以使用带有负索引的`LTRIM`仅保留最近添加的3个元素：

{{< clients-example list_tutorial ltrim_end_of_list >}}
> RPUSH bikes:repairs bike:1 bike:2 bike:3 bike:4 bike:5
(integer) 5
> LTRIM bikes:repairs -3 -1
OK
> LRANGE bikes:repairs 0 -1
1) "bike:3"
2) "bike:4"
3) "bike:5"
{{< /clients-example >}}

以上组合将添加新元素，并将最新的3个元素保留到列表中。使用`LRANGE`您可以访问顶部项目，而无需记住非常旧的数据。

注意：虽然`LRANGE`在技术上是一个O(N)的命令，但访问列表头部或尾部的小范围是一个恒定时间的操作。

屏蔽列表上的操作
---

列表具有一种特殊的特性，使它们适用于实现队列，并且通常作为进程间通信系统的构建块：阻塞操作。

想象一下，你想用一个进程将项目推入一个列表，然后使用另一个进程来对这些项目进行实际工作。这是常见的生产者/消费者设置，并且可以通过以下简单方式实现：

* 将项目推送到列表中，生产者调用`LPUSH`。
* 从列表中提取/处理项目，消费者调用`RPOP`。

然而，有时列表可能为空，并且没有任何内容可处理，因此 `RPOP` 只会返回 NULL。在这种情况下，消费者必须等待一段时间，并使用 `RPOP` 重试。这被称为*轮询*，在这种情况下并不是一个好主意，因为它有几个缺点：

该操作会强制Redis和客户端处理无用的命令（当列表为空时，所有的请求都不会执行任何实际的操作，只会返回NULL）。
此外，它还会延迟处理项目的时间，因为在一个工作者收到NULL后，会等待一段时间。为了减小延迟，我们可以在调用`RPOP`之间的等待时间减少，这会放大问题1的效果，即更多无用的Redis调用。

Redis实现了称为`BRPOP`和`BLPOP`的命令，它们是`RPOP`和`LPOP`的版本，
如果列表为空，它们可以阻塞：只有当向列表添加了新元素或达到用户指定的超时时，
它们才会返回给调用者。

这是我们在工作进程中可以使用的`BRPOP`调用的示例：

{{< clients-example list_tutorial brpop >}}
> RPUSH bikes:repairs bike:1 bike:2
(integer) 5
> BRPOP bikes:repairs 1
1) "bikes:repairs"
2) "bike:2"
> BRPOP bikes:repairs 1
1) "bikes:repairs"
2) "bike:1"
> BRPOP bikes:repairs 1
(nil)
(2.01s)
{{< /clients-example >}}

它的意思是：等待列表`bikes:repairs`中的元素，但如果5秒后没有可用元素，立即返回。

请注意，您可以使用0作为超时时间来永远等待元素，并且您还可以指定多个列表而不仅仅是一个，以便同时等待多个列表，当第一个列表接收到元素时会收到通知。

关于 `BRPOP` 需要注意以下几点：

1. 客户端按顺序提供服务：被阻塞等待列表的第一个客户端在其他客户端推入元素时被首先服务，以此类推。
2. 返回值与 `RPOP` 不同：它是一个包含两个元素的数组，因为 `BRPOP` 和 `BLPOP` 能够阻塞等待来自多个列表的元素，并且还包括键名。
3. 如果达到了超时时间，将返回 NULL。

关于列表和阻塞操作，还有更多的事情您需要了解。我们建议您进一步阅读以下内容：

* 使用 `LMOVE` 可以构建更安全的队列或旋转队列。
* 该命令还有一个阻塞变种，称为 `BLMOVE`。

## 自动创建和删除密钥

到目前为止，在我们的示例中，我们从未在推送元素之前创建空列表，也从未在列表不再包含元素时删除空列表。
当列表留空时，Redis会负责删除键，或者在键不存在且我们尝试添加元素时创建空列表，例如使用`LPUSH`命令。

该文本不仅适用于列表，也适用于Redis的所有多元素数据类型--流、集合、有序集合和哈希。

基本上，我们可以用三条规则来总结行为：

1. 当我们将元素添加到聚合数据类型中时，如果目标键不存在，则在添加元素之前会创建一个空的聚合数据类型。
2. 当我们从聚合数据类型中移除元素时，如果值仍为空，则键会自动销毁。流数据类型是唯一的例外。
3. 调用只读命令，例如返回列表长度的 `LLEN` 命令，或者删除元素的写命令，对于空键总是产生与预期的聚合类型为空的键相同的结果。

{{< clients-example list_tutorial rule_1 >}}
> DEL new_bikes
(integer) 1
> LPUSH new_bikes bike:1 bike:2 bike:3
(integer) 3
{{< /clients-example >}}

然而，如果存在该键，我们无法对错误的类型执行操作：

{{< clients-example list_tutorial rule_1.1 >}}
> SET new_bikes bike:1
OK
> TYPE new_bikes
string
> LPUSH new_bikes bike:2 bike:3
(error) WRONGTYPE Operation against a key holding the wrong kind of value
{{< /clients-example >}}

第二条规则的示例：

{{< clients-example list_tutorial rule_2 >}}
> RPUSH bikes:repairs bike:1 bike:2 bike:3
(integer) 3
> EXISTS bikes:repairs
(integer) 1
> LPOP bikes:repairs
"bike:3"
> LPOP bikes:repairs
"bike:2"
> LPOP bikes:repairs
"bike:1"
> EXISTS bikes:repairs
(integer) 0
{{< /clients-example >}}

所有元素弹出后，键不再存在。

规则3的例子：

{{< clients-example list_tutorial rule_3 >}}
> DEL bikes:repairs
(integer) 0
> LLEN bikes:repairs
(integer) 0
> LPOP bikes:repairs
(nil)
{{< /clients-example >}}


## 限制

Redis列表的最大长度是2^32 - 1（4,294,967,295）个元素。


## 性能

访问列表的头部或尾部的列表操作为O(1)，这意味着它们非常高效。
然而，用于操作列表中元素的命令通常是O(n)。
其中的示例包括`LINDEX`，`LINSERT`和`LSET`。
在运行这些命令时要谨慎，尤其是在操作大型列表时。

## 替代方案

当你需要存储和处理一个不确定的事件序列时，将Redis Streams视为列表的一个替代方案。

## 了解更多

* [Redis列表详解](https://www.youtube.com/watch?v=PB5SeOkkxQc) 是一段简短而全面的Redis列表解释视频。
* [Redis大学的RU101](https://university.redis.com/courses/ru101/)详细介绍了Redis列表。
