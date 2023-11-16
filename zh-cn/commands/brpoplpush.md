`BRPOPLPUSH` 是 `RPOPLPUSH` 的阻塞形式。
当 `source` 包含元素时，这个命令的行为与 `RPOPLPUSH` 完全一致。
当在 `MULTI`/`EXEC` 块中使用时，这个命令的行为与 `RPOPLPUSH` 完全一致。
当 `source` 为空时，Redis 将会阻塞连接直到其他客户端向其推送或达到 `timeout`。
可以使用零值的 `timeout` 来无限期阻塞。

有关更多信息，请参阅 `RPOPLPUSH`。

## 模式：可靠队列

请查看`RPOPLPUSH`文档中的模式描述。

## 模式：循环列表

请在 `RPOPLPUSH` 文档中查看模式说明。
