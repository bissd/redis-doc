`CLUSTER INFO`提供关于Redis Cluster重要参数的`INFO`样式信息。
回复中始终包含以下字段：

```
cluster_state:ok
cluster_slots_assigned:16384
cluster_slots_ok:16384
cluster_slots_pfail:0
cluster_slots_fail:0
cluster_known_nodes:6
cluster_size:3
cluster_current_epoch:6
cluster_my_epoch:2
cluster_stats_messages_sent:1483972
cluster_stats_messages_received:1483968
total_cluster_links_buffer_limit_exceeded:0
```

* `cluster_state`: 如果节点能接收查询，则状态为 `ok`。如果至少有一个哈希槽未绑定（没有关联节点），处于错误状态（用 FAIL 标志标记的节点正在提供服务），或者如果大多数主节点无法被该节点访问，则状态为 `fail`。
* `cluster_slots_assigned`: 关联到某个节点的哈希槽数量（未绑定的除外）。节点正常工作应该有 16384 个哈希槽，这意味着每个哈希槽应该映射到一个节点。
* `cluster_slots_ok`: 映射到不处于 `FAIL` 或 `PFAIL` 状态的节点的哈希槽数量。
* `cluster_slots_pfail`: 映射到 `PFAIL` 状态的节点的哈希槽数量。请注意，只要故障检测算法未将 `PFAIL` 状态提升为 `FAIL` 状态，这些哈希槽仍然正常工作。`PFAIL` 只意味着我们目前无法与节点通讯，但可能只是一个临时错误。
* `cluster_slots_fail`: 映射到 `FAIL` 状态的节点的哈希槽数量。如果此数字不为零，则节点无法提供查询，除非在配置中将 `cluster-require-full-coverage` 设置为 `no`。
* `cluster_known_nodes`: 集群中已知节点的总数，包括当前可能不是集群的正确成员的 `HANDSHAKE` 状态的节点。
* `cluster_size`: 集群中至少为一个哈希槽提供服务的主节点数量。
* `cluster_current_epoch`: 本地的 `Current Epoch` 变量。用于在故障切换期间创建唯一递增的版本号。
* `cluster_my_epoch`: 我们正在通讯的节点的 `Config Epoch`。这是指派给此节点的当前配置版本。
* `cluster_stats_messages_sent`: 通过集群节点之间的二进制总线发送的消息数量。
* `cluster_stats_messages_received`: 通过集群节点之间的二进制总线接收的消息数量。
* `total_cluster_links_buffer_limit_exceeded`: 超过 `cluster-link-sendbuf-limit` 配置而释放的集群连接计数。

如果值不为0，则回复中可能包含以下与消息相关的字段：
每种消息类型都包括发送和接收消息数量的统计信息。
以下是这些字段的解释：

* `cluster_stats_messages_ping_sent`和`cluster_stats_messages_ping_received`：集群总线PING（不要与客户端命令`PING`混淆）。
* `cluster_stats_messages_pong_sent`和`cluster_stats_messages_pong_received`：PONG（对PING的回复）。
* `cluster_stats_messages_meet_sent`和`cluster_stats_messages_meet_received`：通过流言或`CLUSTER MEET`向新节点发送的握手消息。
* `cluster_stats_messages_fail_sent`和`cluster_stats_messages_fail_received`：将节点XXX标记为失败。
* `cluster_stats_messages_publish_sent`和`cluster_stats_messages_publish_received`：发布/订阅发布传播，请参阅[发布/订阅]（/topics/pubsub#pubsub）。
* `cluster_stats_messages_auth-req_sent`和`cluster_stats_messages_auth-req_received`：予以副本初始的领导者选举以替代其主节点。
* `cluster_stats_messages_auth-ack_sent`和`cluster_stats_messages_auth-ack_received`：指示领导者选举中的投票的消息。
* `cluster_stats_messages_update_sent`和`cluster_stats_messages_update_received`：另一个节点的插槽配置。
* `cluster_stats_messages_mfstart_sent`和`cluster_stats_messages_mfstart_received`：暂停客户端进行手动故障转移。
* `cluster_stats_messages_module_sent`和`cluster_stats_messages_module_received`：模块集群API消息。
* `cluster_stats_messages_publishshard_sent`和`cluster_stats_messages_publishshard_received`：发布/订阅发布片段传播，请参阅[分片发布/订阅]（/topics/pubsub#sharded-pubsub）。

关于当前时代(Current Epoch)和配置时代(Config Epoch)变量的更多信息，请参考[Redis Cluster specification document](/topics/cluster-spec#cluster-current-epoch)。
