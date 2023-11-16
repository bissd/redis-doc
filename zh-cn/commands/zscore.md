返回`key`中排序集合中`member`的分数。

如果在有序集合中找不到`member`，或者`key`不存在，则会返回`nil`。

@examples

```cli
ZADD myzset 1 "one"
ZSCORE myzset "one"
```
