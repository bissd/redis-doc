返回所有指定键的值。
对于不包含字符串值或不存在的每个键，返回特殊值`nil`。
因此，该操作永远不会失败。

@examples

```cli
SET key1 "Hello"
SET key2 "World"
MGET key1 key2 nonexisting
```
