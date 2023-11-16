如果 `key` 存在，则返回结果。

用户应该注意，如果同一个现有键在参数中多次被提及，它将被计算多次。因此，如果`somekey`存在，`EXISTS somekey somekey`将返回2。

@examples

```cli
SET key1 "Hello"
EXISTS key1
EXISTS nosuchkey
SET key2 "World"
EXISTS key1 key2 nosuchkey
```
