将指定字段的值设置为存储在“key”中的哈希表中的相应值。

该命令会覆盖哈希中存在的指定字段的值。
如果`key`不存在，则创建一个新的持有哈希的键。

```cli
HSET myhash field1 "Hello"
HGET myhash field1
HSET myhash field2 "Hi" field3 "World"
HGET myhash field2
HGET myhash field3
HGETALL myhash
```
