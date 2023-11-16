该命令返回每个`member`是否是存储在`key`中的集合的成员。

对于每个`member`，如果该值是集合的成员，则返回`1`，如果元素不是集合的成员或者`key`不存在，则返回`0`。

@examples

```cli
SADD myset "one"
SADD myset "one"
SMISMEMBER myset "one" "notamember"
```
