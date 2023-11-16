返回存储在`key`中的哈希的所有字段和值。
在返回的值中，每个字段名称后面跟着它的值，因此回复的长度是哈希大小的两倍。

@examples

```cli
HSET myhash field1 "Hello"
HSET myhash field2 "World"
HGETALL myhash
```
