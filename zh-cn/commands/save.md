`SAVE` 命令执行**同步**保存数据集的操作，将 Redis 实例中的所有数据作为一个在时间点上的快照，以 RDB 文件的形式保存。

几乎从不在生产环境中调用 `SAVE` 命令，因为它会阻塞所有其他客户端。
通常使用 `BGSAVE` 命令。
然而，如果出现问题导致 Redis 无法创建后台保存子进程（例如 `fork(2)` 系统调用错误），`SAVE` 命令可以作为最后的手段，执行最新数据集的转储。

请参考[persistence documentation][tp]以获取详细信息。

[tp]: /topics/persistence
