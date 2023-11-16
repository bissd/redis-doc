计算第一个输入排序集合和所有后续输入之间的差异，并将结果存储在`destination`中。输入键的总数由`numkeys`指定。

不存在的键被视为空集合。

如果`destination`已存在，则会被覆盖。

@examples

```cli
ZADD zset1 1 "one"
ZADD zset1 2 "two"
ZADD zset1 3 "three"
ZADD zset2 1 "one"
ZADD zset2 2 "two"
ZDIFFSTORE out 2 zset1 zset2
ZRANGE out 0 -1 WITHSCORES
```
