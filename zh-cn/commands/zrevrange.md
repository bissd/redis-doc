返回存储在“key”中的有序集合中指定范围的元素。
元素被认为从最高分到最低分排序。
对于具有相等分数的元素，使用降序字典排序。

除了排序顺序相反外，`ZREVRANGE`与`ZRANGE`类似。

@examples

```cli
ZADD myzset 1 "one"
ZADD myzset 2 "two"
ZADD myzset 3 "three"
ZREVRANGE myzset 0 -1
ZREVRANGE myzset 2 3
ZREVRANGE myzset -2 -1
```
