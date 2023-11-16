从存储在 `key` 中的已排序集合中移除并返回最低分数的 `count` 个成员。

当未指定时，`count` 的默认值为 1。指定一个比排序集的基数更高的 `count` 值将不会产生错误。在返回多个元素时，具有最低分数的元素将首先出现，然后是分数更高的元素。

@examples

```cli
ZADD myzset 1 "one"
ZADD myzset 2 "two"
ZADD myzset 3 "three"
ZPOPMIN myzset
```
