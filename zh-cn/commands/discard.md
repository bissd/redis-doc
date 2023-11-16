将 [transaction][tt] 中先前排队的所有命令刷新，并将连接状态恢复为正常。

[tt]: /topics/transactions

如果使用了`WATCH`命令，那么使用`DISCARD`命令将取消连接所监视的所有键的监视状态。
