从存储在`key`中的哈希中删除指定字段。
忽略在哈希中不存在的指定字段。
如果`key`不存在，它被视为一个空哈希，并且此命令返回`0`。

@examples

```cli
HSET myhash field1 "foo"
HDEL myhash field1
HDEL myhash field2
```
