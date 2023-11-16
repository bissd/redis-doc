以`key`为键存储的有序集合中，移除所有分数介于`min`和`max`之间（包括`min`和`max`）的元素。

@examples

```cli
ZADD myzset 1 "one"
ZADD myzset 2 "two"
ZADD myzset 3 "three"
ZREMRANGEBYSCORE myzset -inf (2
ZRANGE myzset 0 -1 WITHSCORES
```
