以原子方式将`key`设置为`value`，并返回存储在`key`中的旧值。
当`key`存在但不包含字符串时返回错误。在成功的`SET`操作中，任何先前关联到该键的生存时间都被丢弃。

## 设计模式

`GETSET`可以与`INCR`一起使用来进行计数和原子重置。
例如：一个进程可能会对键`mycounter`调用`INCR`，每当发生某个事件时，
但有时我们需要获取计数器的值并原子性地将其重置为零。
可以使用`GETSET mycounter "0"`来完成这个操作：

```cli
INCR mycounter
GETSET mycounter "0"
GET mycounter
```

@examples

```cli
SET mykey "Hello"
GETSET mykey "World"
GET mykey
```
