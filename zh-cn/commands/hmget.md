返回在存储在`key`中的哈希中与指定`fields`相关联的值。

对于哈希中不存在的每个字段，将返回一个`nil`值。
因为不存在的键被视为空哈希，对一个不存在的键运行`HMGET`将返回一个`nil`值列表。

```cli
HSET myhash field1 "Hello"
HSET myhash field2 "World"
HMGET myhash field1 field2 nofield
```
