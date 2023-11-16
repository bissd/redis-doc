以“key”存储的有序集合中，删除所有在“start”和“stop”之间的元素。
“start”和“stop”是基于0的索引，其中0表示具有最低分数的元素。
这些索引可以是负数，表示从具有最高分数的元素开始的偏移量。
例如：-1是具有最高分数的元素，-2是具有第二高分数的元素，依此类推。

@examples

```cli
ZADD myzset 1 "one"
ZADD myzset 2 "two"
ZADD myzset 3 "three"
ZREMRANGEBYRANK myzset 0 1
ZRANGE myzset 0 -1 WITHSCORES
```
