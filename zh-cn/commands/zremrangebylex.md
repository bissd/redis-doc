在有序集中，当所有元素都以相同的分数插入时，为了强制按字母顺序排序，此命令会删除存储在`key`中按字母顺序范围由`min`和`max`指定的有序集合中的所有元素。

`min`和`max`的含义与`ZRANGEBYLEX`命令相同。同样地，如果使用相同的`min`和`max`参数调用`ZRANGEBYLEX`命令，此命令实际上会移除与其返回的相同元素。

@examples

```cli
ZADD myzset 0 aaaa 0 b 0 c 0 d 0 e
ZADD myzset 0 foo 0 zap 0 zip 0 ALPHA 0 alpha
ZRANGE myzset 0 -1
ZREMRANGEBYLEX myzset [alpha [omega
ZRANGE myzset 0 -1
```
