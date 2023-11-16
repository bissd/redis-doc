在存储在`key`中的有序集合中，通过`increment`增加`member`的分数。
如果有序集合中不存在`member`，则将其添加，并将分数设置为`increment`（就像其之前的分数是`0.0`一样）。
如果`key`不存在，则创建一个新的有序集合，其中包含指定的`member`作为唯一成员。

当`key`存在但不是有序集合时，会返回一个错误。

`score`值应为数字值的字符串表示，接受双精度浮点数。
可以使用负值来减少分数。

@examples

```cli
ZADD myzset 1 "one"
ZADD myzset 2 "two"
ZINCRBY myzset 2 "one"
ZRANGE myzset 0 -1 WITHSCORES
```
