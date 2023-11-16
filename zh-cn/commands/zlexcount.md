当排序集合中的所有元素都以相同的分数插入时，为了强制字典顺序，此命令返回 `key` 中值介于 `min` 和 `max` 之间的排序集合中的元素数量。

`min`和`max`参数的含义与`ZRANGEBYLEX`中所描述的相同。

注意：该命令的复杂度仅为O(log(N))，因为它使用了元素的排名（参见`ZRANK`）来获取范围的概念。因此，无需执行与范围大小成正比的工作量。

@examples

```cli
ZADD myzset 0 a 0 b 0 c 0 d 0 e
ZADD myzset 0 f 0 g
ZLEXCOUNT myzset - +
ZLEXCOUNT myzset [b [f
```
