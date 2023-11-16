计算由指定键给出的`numkeys`个有序集合的并集，并将结果存储在`destination`中。
在传递输入键和其他（可选）参数之前，必须提供输入键的数量（`numkeys`）。

默认情况下，元素的最终分数是其在存在的有序集合中分数的总和。

使用`WEIGHTS`选项，可以为每个输入的有序集合指定一个乘法因子。
这意味着在传递给聚合函数之前，每个输入的有序集合中的每个元素的分数都会乘以这个因子。
当未给出`WEIGHTS`时，乘法因子默认为`1`。

通过使用“AGGREGATE”选项，可以指定联合结果的聚合方式。
该选项默认为“SUM”，其中元素的分数将在存在的输入中进行求和。
当此选项设置为“MIN”或“MAX”时，生成的集合将包含元素在存在的输入中的最小或最大分数。

如果目标已经存在，则会被覆盖。

@examples

```cli
ZADD zset1 1 "one"
ZADD zset1 2 "two"
ZADD zset2 1 "one"
ZADD zset2 2 "two"
ZADD zset2 3 "three"
ZUNIONSTORE out 2 zset1 zset2 WEIGHTS 2 3
ZRANGE out 0 -1 WITHSCORES
```
