将指定的字段设置为存储在`key`中的哈希表中的相应值。
此命令将覆盖哈希表中已存在的任何指定字段。
如果`key`不存在，则会创建一个新的持有哈希表的键。

@examples

```cli
HMSET myhash field1 "Hello" field2 "World"
HGET myhash field1
HGET myhash field2
```
