计算由指定键给出的`numkeys`个已排序集合的交集，并将结果存储在`destination`中。
在传递输入键和其他（可选）参数之前，必须提供输入键的数量（`numkeys`）。

默认情况下，元素的结果分数是其存在于排序集合中的分数之和。
因为交集要求元素成为每个给定排序集合的成员，所以结果排序集合中每个元素的分数都等于输入排序集合的数量。

对于`WEIGHTS`和`AGGREGATE`选项的描述，请参阅`ZUNIONSTORE`。

如果`目标`已经存在，则会被覆盖。

@examples

```cli
ZADD zset1 1 "one"
ZADD zset1 2 "two"
ZADD zset2 1 "one"
ZADD zset2 2 "two"
ZADD zset2 3 "three"
ZINTERSTORE out 2 zset1 zset2 WEIGHTS 2 3
ZRANGE out 0 -1 WITHSCORES
```
