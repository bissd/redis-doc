从存储在“key”中的集合中删除指定的成员。
忽略不是该集合成员的指定成员。
如果“key”不存在，则将其视为空集合，并返回此命令返回的值为 `0`。

当存储在`key`的值不是一个集合时，将返回错误。

@examples

```cli
SADD myset "one"
SADD myset "two"
SADD myset "three"
SREM myset "one"
SREM myset "four"
SMEMBERS myset
```
