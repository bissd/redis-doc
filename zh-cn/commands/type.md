返回存储在`key`上值的类型的字符串表示。
可能返回的不同类型包括：`string`（字符串）、`list`（列表）、`set`（集合）、`zset`（有序集合）、`hash`（哈希表）和`stream`（流）。

@examples

```cli
SET key1 "value"
LPUSH key2 "value"
SADD key3 "value"
TYPE key1
TYPE key2
TYPE key3
```
