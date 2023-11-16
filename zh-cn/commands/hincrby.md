以“key”存储的哈希中的“field”的数字递增“increment”。
如果“key”不存在，则创建一个新的持有哈希的键。
如果“field”不存在，则在执行操作之前将其值设置为“0”。

`HINCRBY` 支持的值的范围限制在 64 位有符号整数内。

@examples

由于`increment`参数是有符号的，因此可以进行增加和减少操作：

```cli
HSET myhash field 5
HINCRBY myhash field 1
HINCRBY myhash field -1
HINCRBY myhash field -10
```
