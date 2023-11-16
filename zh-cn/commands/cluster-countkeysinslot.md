返回指定 Redis 集群哈希槽中的键数。该命令仅查询本地数据集，因此与不服务于指定哈希槽的节点进行联系将始终返回计数为零。

```
> CLUSTER COUNTKEYSINSLOT 7000
(integer) 50341
```
