这个`CLUSTER DELSLOTSRANGE`命令与`CLUSTER DELSLOTS`命令相似，它们都可以从节点上删除哈希槽。
不同的是，`CLUSTER DELSLOTS`命令需要提供一个要从节点上删除的哈希槽的列表，而`CLUSTER DELSLOTSRANGE`命令需要提供一个要从节点上删除的哈希槽范围的列表（由开始和结束的槽位指定）。

## 示例

要从节点中删除槽位1、2、3、4和5，可以使用`CLUSTER DELSLOTS`命令：

    > CLUSTER DELSLOTS 1 2 3 4 5
    OK

可以使用以下`CLUSTER DELSLOTSRANGE`命令完成相同的操作：

    > CLUSTER DELSLOTSRANGE 1 5
    OK

但是，请注意：

1. 只有当所有指定的槽已经与节点关联时，该命令才起作用。
2. 如果指定了同一个槽多次，则该命令将失败。
3. 由于未覆盖所有哈希槽，命令执行的副作用可能导致节点进入“down”状态。

## Redis 集群中的用法

此命令仅适用于集群模式，并且可能在调试和手动编排集群配置时非常有用，特别是在创建新集群时。目前 `redis-cli` 不使用该命令，主要是为了 API 的完整性而存在。
