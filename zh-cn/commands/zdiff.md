该命令与`ZDIFFSTORE`类似，但不是将结果存储到排序集合中，而是返回给客户端。

以下是示例：

@examples

```cli
ZADD zset1 1 "one"
ZADD zset1 2 "two"
ZADD zset1 3 "three"
ZADD zset2 1 "one"
ZADD zset2 2 "two"
ZDIFF 2 zset1 zset2
ZDIFF 2 zset1 zset2 WITHSCORES
```
