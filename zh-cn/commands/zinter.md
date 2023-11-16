此命令类似于`ZINTERSTORE`，但不是将结果存储在有序集合中，而是返回给客户端。

对于`WEIGHTS`和`AGGREGATE`选项的说明，请参阅`ZUNIONSTORE`。

@examples

```cli
ZADD zset1 1 "one"
ZADD zset1 2 "two"
ZADD zset2 1 "one"
ZADD zset2 2 "two"
ZADD zset2 3 "three"
ZINTER 2 zset1 zset2
ZINTER 2 zset1 zset2 WITHSCORES
```
