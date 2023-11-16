返回存储在“key”键下的集合值的所有成员。

这与使用一个参数`key`运行的`SINTER`具有相同的效果。

以下是一些示例：

```cli
SADD myset "Hello"
SADD myset "World"
SMEMBERS myset
```
