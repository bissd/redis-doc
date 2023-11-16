订阅客户端到给定的模式。

支持的 glob-style 模式：

* `h?llo` 订阅 `hello`、`hallo` 和 `hxllo`
* `h*llo` 订阅 `hllo` 和 `heeeello`
* `h[ae]llo` 订阅 `hello` 和 `hallo`，但不包括 `hillo`

如果您想直接匹配特殊字符，可以使用“\”进行转义。

当客户端进入已订阅状态后，除了额外的`SUBSCRIBE`、`SSUBSCRIBE`、`PSUBSCRIBE`、`UNSUBSCRIBE`、`SUNSUBSCRIBE`、`PUNSUBSCRIBE`、`PING`、`RESET`和`QUIT`命令外，不应发出任何其他命令。
然而，如果使用RESP3（参见`HELLO`），客户端可以在订阅状态下发出任何命令。

有关更多信息，请参阅[Pub/sub](/docs/interact/pubsub/)。

## 行为变更历史

*   `>= 6.2.0`：可以调用 `RESET` 方法来退出订阅状态。
