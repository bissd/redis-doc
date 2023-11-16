返回存储在`key`中的有序集合中，与指定的`members`相关联的分数。

对于不存在于有序集合中的每个 `member` ，将返回一个 `nil` 值。

@examples

```cli
ZADD myzset 1 "one"
ZADD myzset 2 "two"
ZMSCORE myzset "one" "two" "nofield"
```
