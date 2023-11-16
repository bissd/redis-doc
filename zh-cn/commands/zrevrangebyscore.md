在键“key”上的排序集中，分数在“max”和“min”之间（包括分数等于“max”或“min”的元素）的所有元素。
与排序集的默认排序相反，对于此命令，元素被认为按分数从高到低排序。

具有相同分数的元素按照逆字典顺序返回。

除了排序方式相反外，`ZREVRANGEBYSCORE`与`ZRANGEBYSCORE`相似。

@examples

```cli
ZADD myzset 1 "one"
ZADD myzset 2 "two"
ZADD myzset 3 "three"
ZREVRANGEBYSCORE myzset +inf -inf
ZREVRANGEBYSCORE myzset 2 1
ZREVRANGEBYSCORE myzset 2 (1
ZREVRANGEBYSCORE myzset (2 (1
```
