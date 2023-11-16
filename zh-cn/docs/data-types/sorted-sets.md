---
title: "Redis有序集合"
linkTitle: "有序集合"
weight: 50
description: >
    Redis 有序集合简介
---

Redis有序集合是一组通过关联分数排序的唯一字符串（成员）的集合。
当多个字符串具有相同的分数时，字符串按字典顺序排序。
一些使用有序集合的用例包括：

* 排行榜。例如，您可以使用有序集合轻松维护大型在线游戏中最高分数的有序列表。
* 速率限制器。特别是，您可以使用有序集合构建滑动窗口速率限制器，以防止过多的API请求。

您可以将有序集合视为Set和Hash的混合体。与Set一样，有序集合由唯一且不重复的字符串元素组成，因此在某种程度上，有序集合也是一个集合。

然而，尽管集合内部的元素没有顺序，但是排序集合中的每个元素都与浮点值关联，被称为*分数*（这也是为什么类型也类似于散列，因为每个元素都映射到一个值）。

此外，有序集合中的元素是*按顺序取出*的（因此它们不是按照请求排序的，排序是用于表示有序集合的数据结构的特性）。它们按照以下规则排序：

- 如果B和A是具有不同分数的两个元素，则如果A.score > B.score，则A > B。
- 如果B和A具有完全相同的分数，则如果A字符串在字典顺序上大于B字符串，则A > B。由于有序集合只有唯一元素，所以B和A字符串不能相等。

让我们从一个简单的例子开始，我们将列出所有参赛选手及他们在第一场比赛中得到的分数：

{{< clients-example ss_tutorial zadd >}}
> ZADD racer_scores 10 "Norem"
(integer) 1
> ZADD racer_scores 12 "Castilla"
(integer) 1
> ZADD racer_scores 8 "Sam-Bodden" 10 "Royce" 6 "Ford" 14 "Prickett"
(integer) 4
{{< /clients-example >}}


正如你所见，`ZADD`类似于`SADD`，但需要一个额外的参数（在要添加的元素之前放置）作为分数。
`ZADD`也是多参数的，所以你可以自由地指定多个分数-值对，即使在上面的示例中没有使用。

使用排序集合非常容易按照黑客的出生年份返回一个已经按照排序的黑客列表，因为实际上*它们已经被排序*。

实现说明：有序集合通过一个双端口的数据结构实现，其中包含了跳表和哈希表。因此，每次添加元素时，Redis都会执行一次O(log(N))的操作。这是很好的，但是当我们要求排序元素时，Redis根本不需要做任何工作，因为它已经是排序好的。请注意，`ZRANGE`命令的排序顺序是从低到高的，而`ZREVRANGE`命令的顺序是从高到低的。


{{< clients-example ss_tutorial zrange >}}
> ZRANGE racer_scores 0 -1
1) "Ford"
2) "Sam-Bodden"
3) "Norem"
4) "Royce"
5) "Castilla"
6) "Prickett"
> ZREVRANGE racer_scores 0 -1
1) "Prickett"
2) "Castilla"
3) "Royce"
4) "Norem"
5) "Sam-Bodden"
6) "Ford"
{{< /clients-example >}}

注意：0和-1表示从元素索引为0到最后一个元素（-1在此处的工作方式与“LRANGE”命令的情况相同）。

可以使用`WITHSCORES`参数来返回分数：

{{< clients-example ss_tutorial zrange_withscores >}}
> ZRANGE racer_scores 0 -1 withscores
1) "Ford"
2) "6"
3) "Sam-Bodden"
4) "8"
5) "Norem"
6) "10"
7) "Royce"
8) "10"
9) "Castilla"
10) "12"
11) "Prickett"
12) "14"
{{< /clients-example >}}

###  操作范围

有序集合比这更强大。它们可以对范围进行操作。
让我们获取所有得分为10或更少的赛车手。我们
使用`ZRANGEBYSCORE`命令来完成：

{{< clients-example ss_tutorial zrangebyscore >}}
> ZRANGEBYSCORE racer_scores -inf 10
1) "Ford"
2) "Sam-Bodden"
3) "Norem"
4) "Royce"
{{< /clients-example >}}

我们要求Redis返回所有分数介于负无穷到10之间的元素（两个极端都包括在内）。

要删除一个元素，我们只需使用`ZREM`并提供赛车手的姓名。
还可以删除一定范围的元素。让我们删除赛车手Castilla以及所有得分少于10分的赛车手。

{{< clients-example ss_tutorial zremrangebyscore >}}
> ZREM racer_scores "Castilla"
(integer) 1
> ZREMRANGEBYSCORE racer_scores -inf 9
(integer) 2
> ZRANGE racer_scores 0 -1
1) "Norem"
2) "Royce"
3) "Prickett"
{{< /clients-example >}}

`ZREMRANGEBYSCORE` 或许不是最好的命令名称，
但它非常有用，且会返回被移除的元素数量。

另一个对于有序集合元素非常有用的操作是获取排名。可以询问一个元素在有序元素集合中的位置。
`ZREVRANK` 命令也可用于获取排名，考虑到元素被按降序排序的方式。

{{< clients-example ss_tutorial zrank >}}
> ZRANK racer_scores "Norem"
(integer) 0
> ZREVRANK racer_scores "Norem"
(integer) 3
{{< /clients-example >}}

### 字典排序分数

在 Redis 2.8 版本中，引入了一个新的特性，允许按字典顺序获取范围内的元素，假设有序集合中的元素都是使用相同的分数插入的（元素使用 C 的 `memcmp` 函数进行比较，所以可以保证没有排序问题，每个 Redis 实例都会返回相同的输出）。

使用词典范围操作的主要命令是 `ZRANGEBYLEX`、`ZREVRANGEBYLEX`、`ZREMRANGEBYLEX` 和 `ZLEXCOUNT`。

例如，让我们再次添加我们的著名黑客列表，但这次所有元素的分数都是零。我们将看到，由于有序集合的排序规则，它们已经按字典顺序排序。我们可以使用`ZRANGEBYLEX`来请求字典范围。

{{< clients-example ss_tutorial zadd_lex >}}
> ZADD racer_scores 0 "Norem" 0 "Sam-Bodden" 0 "Royce" 0 "Castilla" 0 "Prickett" 0 "Ford"
(integer) 3
> ZRANGE racer_scores 0 -1
1) "Castilla"
2) "Ford"
3) "Norem"
4) "Prickett"
5) "Royce"
6) "Sam-Bodden"
> ZRANGEBYLEX racer_scores [A [L
1) "Castilla"
2) "Ford"
{{< /clients-example >}}

范围可以是包含的也可以是不包含的（取决于第一个字符），正无穷和负无穷的字符串分别用`+`和`-`指定。更多信息请参阅文档。

这个功能很重要，因为它允许我们将有序集合作为通用索引使用。例如，如果您想通过一个128位无符号整数参数索引元素，您只需要将元素添加到一个有序集合中，该集合具有相同的分数（例如0），但具有一个16字节的前缀，该前缀由"大端序的128位数字"组成。由于按字典顺序排列的大端序数字（按原始字节顺序）也是按数字顺序排列的，因此您可以请求128位空间的范围，并获取元素值，丢弃前缀。

如果您想在更严谨的演示环境中查看此功能，
请查看[Redis自动补全演示](http://autocomplete.redis.io)。

更新分数：排行榜
---

在切换到下一个主题之前，对于有序集合的最后注意事项。
有序集合的分数可以随时更新。只要对已包含在有序集合中的元素调用 `ZADD`，其分数（和位置）将以 O(log(N)) 时间复杂度进行更新。因此，有序集合非常适合于大量更新的场景。

由于这种特性，常见的使用场景是排行榜。
典型的应用是Facebook游戏，您可以结合按照用户的高分进行排序的能力和获取排名的操作，以显示前N名用户以及用户在排行榜中的排名（例如，“在这里，您是第4932名得分最好的人”）。

## 示例

* 我们可以用有序集合来表示一个排行榜有两种方法。如果我们知道一个选手的新分数，可以直接通过`ZADD`命令进行更新。然而，如果我们想要给现有分数添加点数，可以使用`ZINCRBY`命令。
{{< clients-example ss_tutorial leaderboard >}}
> ZADD racer_scores 100 "Wood"
(integer) 1
> ZADD racer_scores 100 "Henshaw"
(integer) 1
> ZADD racer_scores 150 "Henshaw"
(integer) 0
> ZINCRBY racer_scores 50 "Wood"
"150"
> ZINCRBY racer_scores 50 "Henshaw"
"200"
{{< /clients-example >}}

您将看到，当成员已经存在时，`ZADD` 返回 0（更新分数），而 `ZINCRBY` 返回新的分数。车手 Henshaw 的分数从 100 变为 150，不考虑之前的分数，然后增加了 50 至 200。

## 基本命令

* `ZADD` 将新的成员和关联的分数添加到有序集合中。如果成员已经存在，则更新分数。
* `ZRANGE` 返回有序集合中给定范围内的成员，按照排序顺序排序。
* `ZRANK` 返回提供的成员的排名，假设排序是升序的。
* `ZREVRANK` 返回提供的成员的排名，假设有序集合是降序的。
 
请查看[已排序集合命令的完整列表](https://redis.io/commands/?group=sorted-set)。

## 性能

大多数排序集操作的时间复杂度为O(log(n))，其中_n_是成员数量。

在使用`ZRANGE`命令返回大量值（例如数万或更多）时，请谨慎操作。
该命令的时间复杂度为O(log(n) + m)，其中_m_是返回结果的数量。

## 替代品

Redis有序集合有时用于索引其他Redis数据结构。
如果您需要索引和查询数据，请考虑使用[JSON](/docs/stack/json)数据类型和[搜索和查询](/docs/stack/search)功能。

## 了解更多

* [Redis Sorted Sets Explained](https://www.youtube.com/watch?v=MUKlxdBQZ7g) 是一个有趣的介绍Redis中有序集的视频。
* [Redis University的RU101](https://university.redis.com/courses/ru101/) 详细探讨了Redis有序集。


* [Redis有序集合解析](https://www.youtube.com/watch?v=MUKlxdBQZ7g) 是一个有趣的介绍Redis中有序集合的视频。
* [Redis University的RU101](https://university.redis.com/courses/ru101/) 详细探讨了Redis中的有序集合。
