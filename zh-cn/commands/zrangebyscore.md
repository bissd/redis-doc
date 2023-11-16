以 `key` 的有序集合中分值在 `min` 和 `max` 之间的所有元素（包括分值等于 `min` 或 `max` 的元素）返回。
这些元素被视为按照从低到高的分值进行排序。

具有相同评分的元素按字典顺序返回（这是由于Redis中有序集合实现的属性，并不涉及进一步计算）。

可选的`LIMIT`参数可用于仅获取匹配元素的范围（类似于SQL中的_SELECT LIMIT offset, count_）。负的`count`返回从`offset`开始的所有元素。
请注意，如果`offset`很大，则需要遍历排序集合以获取要返回的元素之前的`offset`元素，这可能导致O（N）的时间复杂度。

使用可选的`WITHSCORES`参数，可以使命令返回元素和其分数，而不仅仅是元素本身。
该选项自Redis 2.0版本起可用。

## 互斥区间和无穷

`min` 和 `max` 可以为 `-inf` 和 `+inf`，这样你不需要知道有序集合中最高或最低的分数，就可以获取从某个分数开始或到某个分数结束的所有元素。

默认情况下，由`min`和`max`指定的间隔是闭区间（包含）。
可以通过在分数前加上字符`(`来指定开区间（不包含）。
例如：

```
ZRANGEBYSCORE zset (1 5
```

在`1 < score <= 5`的情况下，将返回所有元素。

```
ZRANGEBYSCORE zset (5 (10
```

将返回所有满足 `5 < score < 10` 条件的元素（5和10 不包括在内）。

@examples

```cli
ZADD myzset 1 "one"
ZADD myzset 2 "two"
ZADD myzset 3 "three"
ZRANGEBYSCORE myzset -inf +inf
ZRANGEBYSCORE myzset 1 2
ZRANGEBYSCORE myzset (1 2
ZRANGEBYSCORE myzset (1 (2
```

## 模式：加权随机选择元素

通常 `ZRANGEBYSCORE` 的使用是为了获取得分在索引整数键范围之内的项目，但是使用该命令还有其他不太明显的功能。

例如，当实现马尔可夫链和其他算法时，常见的问题是从一个集合中随机选择一个元素，但是不同的元素可能具有不同的权重，从而改变了它们被选中的概率。

以下是我们使用该命令来挂载这样一个算法的方法：

假设你有元素A、B和C，它们的权重分别为1、2和3。
你计算权重的总和，即1+2+3=6。

在这一点上，您使用以下算法将所有元素添加到排序集合中：

```
SUM = ELEMENTS.TOTAL_WEIGHT // 6 in this case.
SCORE = 0
FOREACH ELE in ELEMENTS
    SCORE += ELE.weight / SUM
    ZADD KEY SCORE ELE
END
```

这意味着你设置为：

```
A to score 0.16
B to score .5
C to score 1
```

由于涉及近似计算，为了避免将C设置为0.998而不是1，我们只需修改上述算法，确保最后一个得分为1（留给读者作为练习...）。

在这个时候，每次你想要获得一个加权随机元素，
只需计算一个介于0和1之间的随机数（类似于大多数编程语言中的`rand()`函数），所以你可以这样做：

    RANDOM_ELE = ZRANGEBYSCORE key RAND() +inf LIMIT 0 1
