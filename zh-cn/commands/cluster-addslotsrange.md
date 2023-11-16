`CLUSTER ADDSLOTSRANGE`命令与`CLUSTER ADDSLOTS`命令类似，它们都将哈希槽分配给节点。

这两个命令的区别是，`CLUSTER ADDSLOTS` 命令接受一个插槽列表，并将这些插槽分配给节点，而 `CLUSTER ADDSLOTSRANGE` 命令接受一个插槽范围列表（由起始插槽和结束插槽指定），并将这些插槽范围分配给节点。

## 示例

为了为节点分配插槽1、2、3、4、5，`CLUSTER ADDSLOTS`命令如下：

    > CLUSTER ADDSLOTS 1 2 3 4 5
    OK

可以使用以下`CLUSTER ADDSLOTSRANGE`命令完成相同的操作：

    > CLUSTER ADDSLOTSRANGE 1 5
    OK


## 使用Redis Cluster

这个命令只能在集群模式下工作，并且在以下Redis集群操作中非常有用：

1. 要创建一个新的集群，可以使用`CLUSTER ADDSLOTSRANGE`来设置初始的主节点，并将可用的哈希槽分配给它们。
2. 为了修复一个出现某些未分配哈希槽的损坏的集群。
