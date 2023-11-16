从存储在 `key` 的有序集合中删除指定成员。
忽略不存在的成员。

当 `key` 存在但不是有序集合时，会返回一个错误。

@examples

```cli
ZADD myzset 1 "one"
ZADD myzset 2 "two"
ZADD myzset 3 "three"
ZREM myzset "two"
ZRANGE myzset 0 -1 WITHSCORES
```
