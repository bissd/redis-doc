`INFO` 命令以一种简单易读的格式返回有关服务器的信息和统计数据，对于计算机来说易于解析，对于人类来说易于阅读。

可选参数可用于选择特定的信息部分：

* `server`: Redis服务器的基本信息
* `clients`: 客户端连接部分
* `memory`: 内存消耗相关信息
* `persistence`: RDB和AOF相关信息
* `stats`: 总体统计信息
* `replication`: 主/从复制信息
* `cpu`: CPU消耗统计
* `commandstats`: Redis命令统计
* `latencystats`: Redis命令延迟百分位数分布统计
* `sentinel`: Redis Sentinel部分（仅适用于Sentinel实例）
* `cluster`: Redis Cluster部分
* `modules`: 模块部分
* `keyspace`: 数据库相关统计信息
* `errorstats`: Redis错误统计信息

它还可以取以下值：

*   `all`: 返回所有章节（不包括模块生成的章节）
*   `default`: 只返回默认的章节集合
*   `everything`: 包括 `all` 和 `modules`

如果未提供参数，则默认使用`default`选项。

```cli
INFO
```

## 笔记

请注意，根据Redis的版本，某些字段可能已添加或删除。因此，强健的客户端应用程序应通过跳过未知属性来解析此命令的结果，并优雅地处理缺少的字段。

以下是 Redis >= 2.4 的字段描述：


以下是**服务器**部分中所有字段的含义：

* `redis_version`：Redis服务器的版本
* `redis_git_sha1`：Git SHA1
* `redis_git_dirty`：Git的dirty标记
* `redis_build_id`：构建ID
* `redis_mode`：服务器的模式（"standalone"，"sentinel"或"cluster"）
* `os`：托管Redis服务器的操作系统
* `arch_bits`：架构（32位或64位）
* `multiplexing_api`：Redis使用的事件循环机制
* `atomicvar_api`：Redis使用的原子变量API
* `gcc_version`：用于编译Redis服务器的GCC编译器的版本
* `process_id`：服务器进程的PID
* `process_supervised`：监督系统（"upstart"，"systemd"，"unknown"或"no"）
* `run_id`：用于Sentinel和Cluster识别Redis服务器的随机值
* `tcp_port`：TCP/IP监听端口
* `server_time_usec`：基于纪元的系统时间，以微秒为精度
* `uptime_in_seconds`：Redis服务器启动以来的秒数
* `uptime_in_days`：以天为单位表示的相同值
* `hz`：服务器当前的频率设置
* `configured_hz`：服务器配置的频率设置
* `lru_clock`：每分钟递增的时钟，用于LRU管理
* `executable`：服务器可执行文件的路径
* `config_file`：配置文件的路径
* `io_threads_active`：指示I/O线程是否活动的标志
* `shutdown_in_milliseconds`：在完成关闭序列之前，副本赶上复制的最长剩余时间。
  仅在关闭期间存在此字段。

以下是**客户端**部分所有字段的含义：

- **name**（名称）：客户的名称
- **contact**（联系人）：客户的联系人
- **email**（电子邮件）：客户的电子邮件地址
- **phone**（电话）：客户的电话号码
- **address**（地址）：客户的地址
- **notes**（备注）：关于客户的附加说明或备注信息

*   `connected_clients`: 客户端连接数（不包括来自副本的连接）
*   `cluster_connections`: 集群总线使用的套接字数量的近似值
*   `maxclients`: `maxclients` 配置指令的值。这是 `connected_clients` 、 `connected_slaves` 和 `cluster_connections` 之和的上限。
*   `client_recent_max_input_buffer`: 当前客户端连接中最大的输入缓冲区
*   `client_recent_max_output_buffer`: 当前客户端连接中最大的输出缓冲区
*   `blocked_clients`: 等待阻塞调用（`BLPOP` 、`BRPOP` 、`BRPOPLPUSH` 、`BLMOVE` 、`BZPOPMIN` 、`BZPOPMAX`）的客户端数量
*   `tracking_clients`: 被跟踪的客户端数量（`CLIENT TRACKING`）
*   `clients_in_timeout_table`: 在客户端超时表中的客户端数量
*   `total_blocking_keys`: 阻塞键的数量。在 Redis 7.2 中添加。
*   `total_blocking_keys_on_nokey`: 当一个或多个客户端希望在键被删除时解除阻塞的阻塞键的数量。在 Redis 7.2 中添加。

以下是**内存**部分中所有字段的含义：

* `used_memory`：Redis使用其分配器

理想情况下，`used_memory_rss`值应该只比`used_memory`稍高一点。
当rss >> used时，较大的差异可能意味着存在（外部的）内存碎片化，可以通过检查`allocator_frag_ratio`和`allocator_frag_bytes`来评估。
当used >> rss时，这意味着部分Redis内存已被操作系统换出：请预期一些显著的延迟。

因为Redis无法控制其分配如何映射到内存页面，所以高`used_memory_rss`常常是内存使用量的剧增的结果。

当Redis释放内存时，内存会被返回给分配器，分配器可能会或可能不会将内存返回给系统。`used_memory`值和操作系统报告的内存消耗可能存在差异。这可能是因为Redis已经使用并释放了内存，但没有将其返回给系统。`used_memory_peak`值通常用于检查这一点。

有关服务器内存的其他自省信息可以通过参考 "MEMORY STATS" 命令和 "MEMORY DOCTOR" 来获取。

以下是`persistence`部分中各个字段的含义：



*   `loading`: 标志，指示转储文件的加载是否正在进行中
*   `async_loading`: 在为旧数据提供服务时，异步加载复制数据集。这意味着`repl-diskless-load`已启用，并且设置为`swapdb`。在Redis 7.0中添加。
*   `current_cow_peak`: 在子进程fork运行时，copy-on-write内存的峰值大小（以字节为单位）
*   `current_cow_size`: 在子进程fork运行时，copy-on-write内存的大小（以字节为单位）
*   `current_cow_size_age`: `current_cow_size`值的年龄，以秒为单位。
*   `current_fork_perc`: 当前fork进程的进度百分比。对于AOF和RDB forks，它是`current_save_keys_processed`与`current_save_keys_total`的百分比。
*   `current_save_keys_processed`: 当前保存操作处理的键数
*   `current_save_keys_total`: 当前保存操作开始时的键数
*   `rdb_changes_since_last_save`: 自上次转储以来的更改次数
*   `rdb_bgsave_in_progress`: 标志，指示RDB保存正在进行中
*   `rdb_last_save_time`: 上次成功RDB保存的基于Epoch的时间戳
*   `rdb_last_bgsave_status`: 上次RDB保存操作的状态
*   `rdb_last_bgsave_time_sec`: 上次RDB保存操作的持续时间（以秒为单位）
*   `rdb_current_bgsave_time_sec`: 如果有，则为进行中的RDB保存操作的持续时间（以秒为单位）
*   `rdb_last_cow_size`: 上次RDB保存操作期间copy-on-write内存的大小（以字节为单位）
*   `rdb_last_load_keys_expired`: 上次RDB加载期间删除的易失性键数。在Redis 7.0中添加。
*   `rdb_last_load_keys_loaded`: 上次RDB加载期间加载的键数。在Redis 7.0中添加。
*   `aof_enabled`: 标志，表示已激活AOF日志记录
*   `aof_rewrite_in_progress`: 标志，指示AOF重写操作正在进行中
*   `aof_rewrite_scheduled`: 标志，表示一旦正在进行的RDB保存完成，将计划进行AOF重写操作。
*   `aof_last_rewrite_time_sec`: 上次AOF重写操作的持续时间（以秒为单位）
*   `aof_current_rewrite_time_sec`: 如果有，则为进行中的AOF重写操作的持续时间（以秒为单位）
*   `aof_last_bgrewrite_status`: 上次AOF重写操作的状态
*   `aof_last_write_status`: 上次写操作到AOF的状态
*   `aof_last_cow_size`: 上次AOF重写操作期间copy-on-write内存的大小（以字节为单位）
*   `module_fork_in_progress`: 标志，指示模块fork正在进行中
*   `module_fork_last_cow_size`: 上次模块fork操作期间copy-on-write内存的大小（以字节为单位）
*   `aof_rewrites`: 自启动以来执行的AOF重写次数
*   `rdb_saves`: 自启动以来执行的RDB快照次数

`rdb_changes_since_last_save` 是指自上次调用 `SAVE` 或 `BGSAVE` 以来，产生了某种数据改变的操作数量。

如果激活了AOF，将添加以下附加字段：

* `aof_current_size`: AOF当前文件大小
* `aof_base_size`: 最新启动或重写时的AOF文件大小
* `aof_pending_rewrite`: 表示AOF重写操作将在当前RDB保存完成后计划执行
* `aof_buffer_length`: AOF缓冲区大小
* `aof_rewrite_buffer_length`: AOF重写缓冲区大小。注意，此字段在Redis 7.0中已被移除
* `aof_pending_bio_fsync`: 后台I/O队列中待处理的fsync作业数量
* `aof_delayed_fsync`: 延迟的fsync计数器

如果有负载操作正在进行，将会添加以下附加字段：

* `loading_start_time`: 载入操作开始时的基于Epoch的时间戳
* `loading_total_bytes`: 总文件大小
* `loading_rdb_used_mem`: 在生成RDB文件时服务器的内存使用量
* `loading_loaded_bytes`: 已加载的字节数
* `loading_loaded_perc`: 以百分比表示的相同值
* `loading_eta_seconds`: 载入完成所需的剩余时间（以秒为单位）

以下是**统计**部分中所有字段的含义：



*   `total_connections_received`：服务器接受的连接总数
*   `total_commands_processed`：服务器处理的命令总数
*   `instantaneous_ops_per_sec`：每秒处理的命令数
*   `total_net_input_bytes`：从网络读取的总字节数
*   `total_net_output_bytes`：写入网络的总字节数
*   `total_net_repl_input_bytes`：为复制目的从网络读取的总字节数
*   `total_net_repl_output_bytes`：为复制目的写入网络的总字节数
*   `instantaneous_input_kbps`：每秒网络的读取速率（KB/秒）
*   `instantaneous_output_kbps`：每秒网络的写入速率（KB/秒）
*   `instantaneous_input_repl_kbps`：每秒复制目的网络的读取速率（KB/秒）
*   `instantaneous_output_repl_kbps`：每秒复制目的网络的写入速率（KB/秒）
*   `rejected_connections`：因为 `maxclients` 限制而被拒绝的连接数
*   `sync_full`：与副本进行全量重新同步的次数
*   `sync_partial_ok`：接受的部分重新同步请求的次数
*   `sync_partial_err`：拒绝的部分重新同步请求的次数
*   `expired_keys`：过期的键事件总数
*   `expired_stale_perc`：可能已过期键的百分比
*   `expired_time_cap_reached_count`：主动过期周期提前停止的次数
*   `expire_cycle_cpu_milliseconds`：主动过期周期所花费的累计时间
*   `evicted_keys`：因为 `maxmemory` 限制而被驱逐的键数
*   `evicted_clients`：因为 `maxmemory-clients` 限制而被驱逐的客户端数。Redis 7.0 中新增。
*   `total_eviction_exceeded_time`：自服务器启动以来，`used_memory` 大于 `maxmemory` 的总时间，以毫秒为单位
*   `current_eviction_exceeded_time`：`used_memory` 最后一次超过 `maxmemory` 的时间，以毫秒为单位
*   `keyspace_hits`：在主字典中成功查找到键的次数
*   `keyspace_misses`：在主字典中查找键失败的次数
*   `pubsub_channels`：具有客户端订阅的全局发布/订阅通道数
*   `pubsub_patterns`：具有客户端订阅的全局发布/订阅模式数
*   `pubsubshard_channels`：具有客户端订阅的全局发布/订阅分片通道数。Redis 7.0.3 中新增。
*   `latest_fork_usec`：最新一次 fork 操作的持续时间（微秒）
*   `total_forks`：服务器启动以来进行的 fork 操作总数
*   `migrate_cached_sockets`：为 `MIGRATE` 目的而开启的套接字数
*   `slave_expires_tracked_keys`：为过期目的而跟踪的键数（仅适用于可写副本）
*   `active_defrag_hits`：主动碎片整理过程执行的值重新分配次数
* `active_defrag_misses`：主动碎片整理进程中已启动但被中止的值重新分配次数
* `active_defrag_key_hits`：已主动碎片整理的键数
* `active_defrag_key_misses`：未被主动碎片整理进程忽略的键数
* `total_active_defrag_time`：内存碎片超过限制的总时间（毫秒）
* `current_active_defrag_time`：上次内存碎片超过限制后经过的时间（毫秒）
* `tracking_total_keys`：服务器正在追踪的键数
* `tracking_total_items`：正在追踪的项数，即每个键的客户端数之和
* `tracking_total_prefixes`：服务器前缀表中正在追踪的前缀数（仅适用于广播模式）
* `unexpected_error_replies`：意外错误回复的数量，即AOF加载或复制过程中出现的错误类型
* `total_error_replies`：发出的错误回复总数，包括拒绝执行的命令（命令执行前的错误）和执行失败的命令（命令执行中的错误）
* `dump_payload_sanitizations`：转储有效负载深度完整性验证的总次数（参见`sanitize-dump-payload`配置）
* `total_reads_processed`：已处理的读取事件总数
* `total_writes_processed`：已处理的写入事件总数
* `io_threaded_reads_processed`：主线程和I/O线程处理的读取事件数
* `io_threaded_writes_processed`：主线程和I/O线程处理的写入事件数
* `stat_reply_buffer_shrinks`：输出缓冲区收缩的总次数
* `stat_reply_buffer_expands`：输出缓冲区扩展的总次数
* `eventloop_cycles`：事件循环总次数
* `eventloop_duration_sum`：事件循环中的总时间（包括I/O和命令处理）（微秒）
* `eventloop_duration_cmd_sum`：执行命令的总时间（微秒）
* `instantaneous_eventloop_cycles_per_sec`：每秒的事件循环次数
* `instantaneous_eventloop_duration_usec`：单个事件循环周期的平均时间（微秒）
* `acl_access_denied_auth`：身份验证失败的次数
* `acl_access_denied_cmd`：由于对命令的访问被拒绝而拒绝的命令数
* `acl_access_denied_key`：由于对键的访问被拒绝而拒绝的命令数
* `acl_access_denied_channel`：由于对频道的访问被拒绝而拒绝的命令数

以下是**复制**部分中所有字段的含义：

- `role`：指示服务器在复制中的角色（主服务器或从服务器）。
- `state`：当前复制的状态。常见的状态有`running`（正在复制中）、`stopped`（已停止）、`error`（出现错误）。
- `lag`：从服务器相对主服务器的复制延迟，以秒为单位。
- `upstream`：从服务器正在复制的主服务器的地址。
- `downstream`：主服务器正在复制到的从服务器的地址。
- `last_heartbeat_time`：最近一次心跳接收的时间戳。
- `last_heartbeat_info`：最近一次心跳接收到的详细信息。
- `last_replication_time`：最近一次成功完成复制的时间戳。
- `last_replication_info`：最近一次复制的详细信息。

*   `role`: 如果实例没有任何副本，则值为"master"，如果实例是某个主实例的副本，则值为"slave"。请注意，一个副本可能是另一个副本的主（链式复制）。
*   `master_failover_state`: 如果正在进行故障切换，则为故障切换的状态。
*   `master_replid`: Redis服务器的复制ID。
*   `master_replid2`: 故障切换后用于PSYNC的辅助复制ID。
*   `master_repl_offset`: 服务器的当前复制偏移量。
*   `second_repl_offset`: 接受复制ID的偏移量。
*   `repl_backlog_active`: 表示复制日志记录正在运行的标志。
*   `repl_backlog_size`: 复制日志记录缓冲区的总大小（以字节为单位）。
*   `repl_backlog_first_byte_offset`: 复制日志记录缓冲区的主偏移量。
*   `repl_backlog_histlen`: 复制日志记录缓冲区中数据的大小（以字节为单位）。

如果实例是一个副本，则提供这些附加字段：

* `master_host`: 主节点的主机名或IP地址
* `master_port`: 主节点监听的TCP端口
* `master_link_status`: 连接状态（上行/断开）
* `master_last_io_seconds_ago`: 自上次与主节点交互的秒数
* `master_sync_in_progress`: 指示主节点正在同步到副本
* `slave_read_repl_offset`: 副本实例的读取复制偏移量
* `slave_repl_offset`: 副本实例的复制偏移量
* `slave_priority`: 实例作为故障转移候选者的优先级
* `slave_read_only`: 标志，指示副本是否为只读
* `replica_announced`: 标志，指示副本是否被Sentinel公告

如果有进行中的同步操作，则提供以下附加字段：

*   `master_sync_total_bytes`: 需要传输的总字节数。当大小未知时，可能为0（例如使用`repl-diskless-sync`配置指令时）。
*   `master_sync_read_bytes`: 已传输的字节数。
*   `master_sync_left_bytes`: 完成同步之前剩余的字节数（当`master_sync_total_bytes`为0时可能为负数）。
*   `master_sync_perc`: `master_sync_read_bytes`占`master_sync_total_bytes`的百分比，或者在`master_sync_total_bytes`为0时使用`loading_rdb_used_mem`进行近似计算。
*   `master_sync_last_io_seconds_ago`: 在SYNC操作期间最后一次传输I/O距离现在的秒数。

如果主节点和副本之间的连接断开，将提供一个额外的字段：

*   `master_link_down_since_seconds`: 自链接断开后的秒数

以下字段始终提供：

*   `connected_slaves`：连接的副本数量

如果服务器配置了`min-slaves-to-write`(或者Redis 5开始用`min-replicas-to-write`)指令，将提供一个附加字段：

* `min_slaves_good_slaves`: 当前被认为是良好副本的数量

对于每个副本，添加以下一行：

*   `slaveXXX`: id，IP地址，端口，状态，偏移量，延迟

以下是**CPU**部分的所有字段的含义：



* `used_cpu_sys`: Redis服务器消耗的系统CPU，即服务器进程的所有线程（主线程和后台线程）消耗的系统CPU之和
* `used_cpu_user`: Redis服务器消耗的用户CPU，即服务器进程的所有线程（主线程和后台线程）消耗的用户CPU之和
* `used_cpu_sys_children`: 后台进程消耗的系统CPU
* `used_cpu_user_children`: 后台进程消耗的用户CPU
* `used_cpu_sys_main_thread`: Redis服务器主线程消耗的系统CPU
* `used_cpu_user_main_thread`: Redis服务器主线程消耗的用户CPU

**commandstats** 部分提供基于命令类型的统计数据，包括达到命令执行（未被拒绝）的调用数量、这些命令消耗的总 CPU 时间、每个命令执行的平均 CPU 消耗、被拒绝调用的数量（命令执行前的错误）以及失败调用的数量（命令执行中的错误）。

对于每种命令类型，添加以下行：

*   `cmdstat_XXX`：`calls=XXX,usec=XXX,usec_per_call=XXX,rejected_calls=XXX,failed_calls=XXX`

**latencystats** 部分提供基于命令类型的延迟百分位分布统计信息。

默认情况下，导出的延迟百分位数为p50、p99和p999。
如果需要更改导出的百分位数，请使用`CONFIG SET latency-tracking-info-percentiles "50.0 99.0 99.9"`。

这部分需要启用扩展延迟监控功能（默认情况下已启用）。
 如果需要启用它，请使用`CONFIG SET latency-tracking yes`。

对于每个命令类型，添加以下行：

*   `latency_percentiles_usec_XXX：p<百分位数 1>=<百分位数 1的值>，p<百分位数 2>=<百分位数 2的值>，...`

**errorstats** 部分可以追踪 Redis 中发生的不同错误，
基于回复错误的前缀（"-"后的第一个单词，直到第一个空格。例如：`ERR`）。

对于每种错误类型，添加以下行:


*   `errorstat_XXX`：`count=XXX`

**哨兵**部分仅适用于Redis哨兵实例。它由以下字段组成：

*   `sentinel_masters`: 此 Sentinel 实例监视的 Redis 主节点数量
*   `sentinel_tilt`: 值为 1 表示此 Sentinel 处于 TILT 模式
*   `sentinel_tilt_since_seconds`: 当前 TILT 持续的秒数，如果未处于 TILT 状态，则为 -1。在 Redis 7.0.0 中添加
*   `sentinel_running_scripts`: 此 Sentinel 当前正在执行的脚本数量
*   `sentinel_scripts_queue_length`: 等待执行的用户脚本队列长度
*   `sentinel_simulate_failure_flags`: `SENTINEL SIMULATE-FAILURE` 命令的标志位
    
**集群**部分目前只包含一个唯一字段：

* `cluster_enabled`: 表示Redis集群已启用

**模块**部分包含了关于已加载模块的附加信息，如果模块提供了相关信息。该部分中属性行的字段部分总是以模块的名称为前缀。

**keyspace** 部分提供了每个数据库的主字典的统计信息。
统计信息包括键的数量以及带有过期时间的键的数量。

对于每个数据库，添加以下行：

*   `dbXXX`： `keys=XXX，expires=XXX`

**调试**部分包含实验性指标，可能会在将来版本中更改或删除。
在调用`INFO`或`INFO ALL`时不会包含该部分，只有在使用`INFO DEBUG`时才会返回该部分。

*   `eventloop_duration_aof_sum`: 在事件循环中以微秒为单位刷新 AOF 所花费的总时间
*   `eventloop_duration_cron_sum`: 在微秒级别中 cron 总共消耗的时间（包括 serverCron 和 beforeSleep，但不包括 IO 和 AOF 刷新）
*   `eventloop_duration_max`: 在单个事件循环周期中所花费的最大时间（以微秒为单位）
*   `eventloop_cmd_per_cycle_max`: 单个事件循环周期中处理的最大命令数量

[**hcgcpgp**]: http://code.google.com/p/google-perftools/

关于在该手册中使用的slave一词的说明：从Redis 5开始，为了向后兼容除外，Redis项目不再使用slave一词。不幸的是，在该命令中，slave一词是协议的一部分，所以只能在该API自然过时之后才能删除这样的出现。

**生成的模块部分**：从Redis 6开始，模块可以将自己的信息注入到`INFO`命令中，默认情况下这些信息被排除在外，即使提供了`all`参数（它将包含加载的模块列表但不包括它们生成的信息字段）。要获取这些信息，您必须使用`modules`参数或`everything`参数。
