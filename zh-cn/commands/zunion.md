这个命令和 `ZUNIONSTORE` 类似，但是不是将结果排序集存储起来，而是返回给客户端。

对于`WEIGHTS`和`AGGREGATE`选项的描述，请参阅`ZUNIONSTORE`。

@examples

```cli
ZADD zset1 1 "one"
ZADD zset1 2 "two"
ZADD zset2 1 "one"
ZADD zset2 2 "two"
ZADD zset2 3 "three"
ZUNION 2 zset1 zset2
ZUNION 2 zset1 zset2 WITHSCORES
```
