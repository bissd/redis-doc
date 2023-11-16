返回流中的条目数。如果指定的键不存在，则命令返回零，就像流是空的一样。
然而要注意，与其他Redis类型不同，流的长度可以为零，因此您应该调用 `TYPE` 或 `EXISTS` 来检查键是否存在。

流在没有条目（例如执行了XDEL命令）后不会自动删除,因为流可能与消费者组有关联。

@examples

```cli
XADD mystream * item 1
XADD mystream * item 2
XADD mystream * item 3
XLEN mystream
```
