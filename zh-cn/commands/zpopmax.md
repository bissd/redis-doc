从存储在`key`的排序集合中删除并返回最高分的`count`个成员。

在未指定的情况下，`count` 的默认值为1。指定比排序集合的基数更高的 `count` 值不会产生错误。在返回多个元素时，得分最高的元素将首先出现，其次是得分较低的元素。

@examples

```cli
ZADD myzset 1 "one"
ZADD myzset 2 "two"
ZADD myzset 3 "three"
ZPOPMAX myzset
```
