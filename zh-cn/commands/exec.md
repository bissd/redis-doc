在 [事务][tt] 中执行所有先前排队的命令，并将连接状态恢复为正常。

[tt]: /topics/transactions

在使用`WATCH`时，只有在所监视的键没有被修改的情况下，`EXEC`命令才会被执行，从而实现了[检查和设置机制][ttc]。

[ttc]: /topics/transactions#cas
