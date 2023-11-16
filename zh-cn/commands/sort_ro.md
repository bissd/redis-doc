`SORT`命令的只读变体。它与原始的`SORT`命令完全相同，但拒绝使用`STORE`选项，并且可以安全地在只读副本中使用。

由于原始的`SORT`命令具有`STORE`选项，它在Redis命令表中被技术上标记为写入命令。因此，在Redis Cluster中的只读副本即使连接处于只读模式（请参见Redis Cluster的`READONLY`命令），它也会将请求重定向到主实例。

为了在只读副本中实现`SORT`的行为而不破坏命令标志的兼容性，引入了`SORT_RO`变体。

请参阅原始的 `SORT` 以获取更多详细信息。

@examples

```
SORT_RO mylist BY weight_*->fieldname GET object_*->fieldname
```
