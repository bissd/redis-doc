返回键为`key`的有序集合中，分数在`min`和`max`之间的元素数量。

`min`和`max`参数的语义与`ZRANGEBYSCORE`所描述的相同。

注意：该命令的复杂度仅为O(log(N))，因为它使用元素的排名（参见 `ZRANK`）来获取范围的概念。因此，无需按范围的大小做与之成比例的工作。

@examples

```cli
ZADD myzset 1 "one"
ZADD myzset 2 "two"
ZADD myzset 3 "three"
ZCOUNT myzset -inf +inf
ZCOUNT myzset (1 3
```
