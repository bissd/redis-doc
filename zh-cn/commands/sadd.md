将指定的成员添加到存储在“key”上的集合中。
已经是该集合成员的指定成员将被忽略。
如果“key”不存在，则在添加指定成员之前将创建一个新的集合。

当存储在 `key` 处的值不是一个集合时，会返回错误。

@examples

```cli
SADD myset "Hello"
SADD myset "World"
SADD myset "World"
SMEMBERS myset
```
