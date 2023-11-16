以指定的增量增加存储在键中的哈希的指定字段，并表示为浮点数。如果增量值为负数，则结果是将哈希字段值减少而不是增加。
如果字段不存在，则在执行操作之前将其设置为`0`。
如果发生以下任何情况之一，将返回错误：

* 字段包含了错误类型的值（不是字符串）。
* 当前字段内容或指定的增量无法解析为双精度浮点数。

该命令的确切行为与`INCRBYFLOAT`命令完全相同，请参考`INCRBYFLOAT`的文档获取更多信息。

@examples

```cli
HSET mykey field 10.50
HINCRBYFLOAT mykey field 0.1
HINCRBYFLOAT mykey field -5
HSET mykey field 5.0e3
HINCRBYFLOAT mykey field 2.0e2
```

## 实现细节

命令始终作为`HSET`操作在复制链路和追加模式文件中传播，以保证底层浮点数运算实现的差异不会导致不一致性。
