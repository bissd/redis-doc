---
title: "模块API参考文档"
linkTitle: "API 参考"
weight: 1
description: >
    Redis模块API参考
aliases:
    - /topics/modules-api-ref
---

<!-- 此文件是使用 utils/generate-module-api-doc.rb 从 module.c 生成的 -->


## 文章节

* [堆分配原生函数](#section-heap-allocation-raw-functions)
* [命令 API](#section-commands-api)
* [模块信息和时间测量](#section-module-information-and-time-measurement)
* [模块的自动内存管理](#section-automatic-memory-management-for-modules)
* [字符串对象 API](#section-string-objects-apis)
* [回复 API](#section-reply-apis)
* [命令复制的 API](#section-commands-replication-api)
* [DB 和键 API-通用 API](#section-db-and-key-apis-generic-api)
* [字符串类型的键 API](#section-key-api-for-string-type)
* [列表类型的键 API](#section-key-api-for-list-type)
* [有序集合类型的键 API](#section-key-api-for-sorted-set-type)
* [有序集合迭代器的键 API](#section-key-api-for-sorted-set-iterator)
* [哈希类型的键 API](#section-key-api-for-hash-type)
* [流类型的键 API](#section-key-api-for-stream-type)
* [从模块调用 Redis 命令](#section-calling-redis-commands-from-modules)
* [模块数据类型](#section-modules-data-types)
* [RDB 加载和保存函数](#section-rdb-loading-and-saving-functions)
* [键摘要 API（调试摘要接口适用于模块类型）](#section-key-digest-api-debug-digest-interface-for-modules-types)
* [模块数据类型的 AOF API](#section-aof-api-for-modules-data-types)
* [IO 上下文处理](#section-io-context-handling)
* [日志](#section-logging)
* [通过模块阻止客户端](#section-blocking-clients-from-modules)
* [线程安全上下文](#section-thread-safe-contexts)
* [模块键空间通知 API](#section-module-keyspace-notifications-api)
* [模块集群 API](#section-modules-cluster-api)
* [模块定时器 API](#section-modules-timers-api)
* [模块事件循环 API](#section-modules-eventloop-api)
* [模块 ACL API](#section-modules-acl-api)
* [模块字典 API](#section-modules-dictionary-api)
* [模块信息字段](#section-modules-info-fields)
* [模块实用 API](#section-modules-utility-apis)
* [模块 API 导出/导入](#section-modules-api-exporting-importing)
* [模块命令过滤器 API](#section-module-command-filter-api)
* [浏览键空间和哈希](#section-scanning-keyspace-and-hashes)
* [模块 fork API](#section-module-fork-api)
* [服务器钩子实现](#section-server-hooks-implementation)
* [模块配置 API](#section-module-configurations-api)
* [RDB 加载/保存 API](#section-rdb-load-save-api)
* [键驱逐 API](#section-key-eviction-api)
* [杂项 API](#section-miscellaneous-apis)
* [碎片整理 API](#section-defrag-api)
* [函数索引](#section-function-index)

<span id="section-heap-allocation-raw-functions"></span>

## 堆分配原始函数

使用这些函数分配的内存会被Redis的键驱逐算法纳入考虑，并在Redis内存使用信息中报告。

<span id="RedisModule_Alloc"></span>

### `RedisModule_Alloc`

`RedisModule_Alloc`函数用于在Redis模块中分配内存。这个函数可以用于分配指定大小的内存块，并返回指向该内存块的指针。需要注意的是，分配的内存块必须由`RedisModule_Free`函数进行释放，不能直接使用`free`函数释放。

#### 函数定义

```c
void *RedisModule_Alloc(size_t bytes);
```

#### 参数

- `bytes`：要分配的内存块的字节数。

#### 返回值

指向分配的内存块的指针，如果分配失败则返回`NULL`。

#### 示例

```c
#include "redismodule.h"

int RedisModule_OnLoad(RedisModuleCtx *ctx, RedisModuleString **argv, int argc) {
    // 在Redis模块中分配10字节大小的内存块
    void *data = RedisModule_Alloc(10);
    if (data == NULL) {
        RedisModule_Log(ctx, "warning", "Failed to allocate memory");
        return REDISMODULE_ERR;
    }

    // 使用分配的内存块进行一些操作

    // 释放分配的内存块
    RedisModule_Free(data);

    return REDISMODULE_OK;
}
```


    void *RedisModule_Alloc(size_t bytes);

**可用自：**4.0.0

使用类似于`malloc()`的方式。使用此函数分配的内存将显示在Redis INFO内存中，并根据maxmemory设置用于键的驱逐，在总体上被视为Redis分配的内存。您应该避免使用`malloc()`。如果无法分配足够的内存，此函数将产生恐慌。

<span id="RedisModule_TryAlloc"></span>

### `RedisModule_TryAlloc`

### `RedisModule_TryAlloc`是一个函数，尝试在Redis模块中分配内存。

    void *RedisModule_TryAlloc(size_t bytes);

**从版本开始可用：**7.0.0

类似于 [`RedisModule_Alloc`](#RedisModule_Alloc)，但在分配失败时返回 NULL，而不是引发错误。

<span id="RedisModule_Calloc"></span>

### `RedisModule_Calloc`
`RedisModule_Calloc` 函数

    void *RedisModule_Calloc(size_t nmemb, size_t size);

**可用自：** 4.0.0

文本格式如下：
类似于`calloc()`使用。使用此函数分配的内存将在Redis INFO memory中报告，根据maxmemory设置用于键驱逐，并且通常会计为Redis分配的内存。您应该避免直接使用`calloc()`。

<span id="RedisModule_Realloc"></span>


### `RedisModule_Realloc`

    void* RedisModule_Realloc(void *ptr, size_t bytes);

**可用版本：**4.0.0

使用类似于`realloc()`的方式，对使用[`RedisModule_Alloc()`](#RedisModule_Alloc)获取的内存进行重新分配。

<span id="RedisModule_Free"></span>

### `RedisModule_Free`

    void RedisModule_Free(void *ptr);

**可用自：** 4.0.0

使用`free()`来释放通过[`RedisModule_Alloc()`](#RedisModule_Alloc)和
[`RedisModule_Realloc()`](#RedisModule_Realloc)获取到的内存。然而，你绝不应该尝试使用[`RedisModule_Free()`](#RedisModule_Free)来释放在你的模块内部使用`malloc()`分配的内存。

<span id="RedisModule_Strdup"></span>

### `RedisModule_Strdup`

    char *RedisModule_Strdup(const char *str);

**可用于版本:** 4.0.0

类似于`strdup()`，但返回由[`RedisModule_Alloc()`](#RedisModule_Alloc)分配的内存。

<span id="RedisModule_PoolAlloc"></span>

### `RedisModule_PoolAlloc`

### `RedisModule_PoolAlloc`

    void *RedisModule_PoolAlloc(RedisModuleCtx *ctx, size_t bytes);

**可用版本：**4.0.0

释放在模块回调函数返回时将自动被释放的堆内存。适用于生命周期短暂且必须在回调返回时释放的小内存分配。如果至少请求了字大小的字节，则返回的内存将与架构字大小对齐，否则它只是对齐到下一个二的幂，因此例如3字节的请求将对齐到4字节，而2字节的请求将对齐到2字节。

由于需要使用池分配器时，没有重新分配的样式函数并不是一个好主意。

如果 `bytes` 为0，则该函数返回NULL。

<span id="section-commands-api"></span>

## 命令 API

这些函数用于实现自定义的Redis命令。

以下为示例，详见[https://redis.io/topics/modules-intro](https://redis.io/topics/modules-intro)。

<span id="RedisModule_IsKeysPositionRequest"></span>

### `RedisModule_IsKeysPositionRequest`

    int RedisModule_IsKeysPositionRequest(RedisModuleCtx *ctx);

**可用自：** 4.0.0

如果以"getkeys-api"标志声明的模块命令以特殊方式调用以获取键位置而不是执行，则返回非零。否则返回零。

<span id="RedisModule_KeyAtPosWithFlags"></span>

### `RedisModule_KeyAtPosWithFlags`
### `RedisModule_KeyAtPosWithFlags`

    void RedisModule_KeyAtPosWithFlags(RedisModuleCtx *ctx, int pos, int flags);

**可用自：**7.0.0

当调用模块命令以获取键的位置时，由于在注册期间被标记为“getkeys-api”，命令实现会使用[`RedisModule_IsKeysPositionRequest()`](#RedisModule_IsKeysPositionRequest) API来检查此特殊调用，并使用此函数来报告键。

支持的标志与`RedisModule_SetCommandInfo`使用的标志相同，请参考`REDISMODULE_CMD_KEY_`*。


以下是一个如何使用的示例：

    if (RedisModule_IsKeysPositionRequest(ctx)) {
        RedisModule_KeyAtPosWithFlags(ctx, 2, REDISMODULE_CMD_KEY_RO | REDISMODULE_CMD_KEY_ACCESS);
        RedisModule_KeyAtPosWithFlags(ctx, 1, REDISMODULE_CMD_KEY_RW | REDISMODULE_CMD_KEY_UPDATE | REDISMODULE_CMD_KEY_ACCESS);
    }

注意：在上面的示例中，获取密钥的API可以通过key-specs（优先）来处理。
只有在无法声明涵盖所有键的密钥规范时，才需要实现getkeys-api。

<span id="RedisModule_KeyAtPos"></span>

### `RedisModule_KeyAtPos`

    void RedisModule_KeyAtPos(RedisModuleCtx *ctx, int pos);

**可用版本：** 4.0.0

该API在[`RedisModule_KeyAtPosWithFlags`](#RedisModule_KeyAtPosWithFlags)被添加之前就存在，现在已经被弃用，但可以用于与旧版本的兼容性，在引入key-specs和flags之前。

<span id="RedisModule_IsChannelsPositionRequest"></span>

### `RedisModule_IsChannelsPositionRequest`
`RedisModule_IsChannelsPositionRequest`

    int RedisModule_IsChannelsPositionRequest(RedisModuleCtx *ctx);

**可用自：**7.0.0

如果以"getchannels-api"标志声明的模块命令以特殊方式调用以获取通道位置而非执行，则返回非零。否则返回零。

<span id="RedisModule_ChannelAtPosWithFlags"></span>

### `RedisModule_ChannelAtPosWithFlags`


    void RedisModule_ChannelAtPosWithFlags(RedisModuleCtx *ctx,
                                           int pos,
                                           int flags);

**可用版本：** 7.0.0

当调用模块命令以获取通道位置时，由于在注册时标记为“getchannels-api”，命令实现会使用 [`RedisModule_IsChannelsPositionRequest()`](#RedisModule_IsChannelsPositionRequest) API 来检查这个特殊调用，并使用该函数报告通道。

支持的标志如下：
- `REDISMODULE_CMD_CHANNEL_SUBSCRIBE`：该命令将订阅频道。
- `REDISMODULE_CMD_CHANNEL_UNSUBSCRIBE`：该命令将取消订阅该频道。
- `REDISMODULE_CMD_CHANNEL_PUBLISH`：该命令将发布至该频道。
- `REDISMODULE_CMD_CHANNEL_PATTERN`：不是在特定频道上操作，而是在由模式指定的任何频道上操作。这是Redis中的PSUBSCRIBE和PUNSUBSCRIBE命令所使用的权限。不适用于PUBLISH权限。

以下是如何使用的示例：

    if (RedisModule_IsChannelsPositionRequest(ctx)) {
        RedisModule_ChannelAtPosWithFlags(ctx, 1, REDISMODULE_CMD_CHANNEL_SUBSCRIBE | REDISMODULE_CMD_CHANNEL_PATTERN);
        RedisModule_ChannelAtPosWithFlags(ctx, 1, REDISMODULE_CMD_CHANNEL_PUBLISH);
    }

注意：声明频道的一个用途是用于评估ACL权限。在此上下文中，取消订阅始终是允许的，因此命令只会根据订阅和发布权限进行检查。这比使用[`RedisModule_ACLCheckChannelPermissions`](#RedisModule_ACLCheckChannelPermissions)更可取，因为它允许在执行命令之前检查ACL权限。

<span id="RedisModule_CreateCommand"></span>

### `RedisModule_CreateCommand`

### `RedisModule_CreateCommand`

    int RedisModule_CreateCommand(RedisModuleCtx *ctx,
                                  const char *name,
                                  RedisModuleCmdFunc cmdfunc,
                                  const char *strflags,
                                  int firstkey,
                                  int lastkey,
                                  int keystep);

**可用版本：**4.0.0

在Redis服务器中注册一个新的命令，将使用RedisModule调用约定通过调用函数指针'cmdfunc'来处理。

```
该函数在以下情况下返回`REDISMODULE_ERR`：
- 在`RedisModule_OnLoad`之外调用创建模块命令的函数。
- 指定的命令已被占用。
- 命令名称包含禁止使用的字符。
- 传递了一组无效的标志。
```

否则返回 `REDISMODULE_OK` 并注册新的命令。

此函数必须在模块的初始化期间调用，位于`RedisModule_OnLoad()`函数内部。在初始化函数之外调用此函数是未定义的。

命令函数类型如下：

     int MyCommand_RedisCommand(RedisModuleCtx *ctx, RedisModuleString **argv, int argc);

并且应始终返回`REDISMODULE_OK`。

'`strflags`'参数集指定命令的行为，应作为由空格分隔的单词组成的C字符串传递，例如"write deny-oom"。该参数集包括以下标志：

* **"write"**：该命令可能会修改数据集（也可能从数据集中读取）。
* **"readonly"**：该命令仅返回键的数据，不进行写操作。
* **"admin"**：该命令为管理命令（可能改变复制或执行类似任务）。
* **"deny-oom"**：该命令可能使用额外内存，在内存不足情况下应该拒绝执行。
* **"deny-script"**：不允许在Lua脚本中使用该命令。
* **"allow-loading"**：允许在服务器加载数据时执行该命令。只有与数据集无关的命令才能在此模式下运行。如不确定，请不要使用此标识。
* **"pubsub"**：该命令会向发布/订阅频道发布数据。
* **"random"**：该命令可能在相同输入参数和键值的情况下产生不同的输出。从Redis 7.0开始，此标识已被弃用。可以使用命令提示声明一个命令为"random"，请参阅https://redis.io/topics/command-tips。
* **"allow-stale"**：该命令允许在不提供过期数据的从服务器上执行。如果不清楚其含义，请勿使用此标识。
* **"no-monitor"**：不在监视器上传播该命令。如果该命令的参数中包含敏感数据，请使用此标识。
* **"no-slowlog"**：不在慢日志中记录该命令。如果该命令的参数中包含敏感数据，请使用此标识。
* **"fast"**：该命令的时间复杂度不大于O(log(N))，其中N为集合的大小或代表命令的正常可扩展性问题的其他内容。
* **"getkeys-api"**：该命令实现了返回作为键的参数的接口。当由于命令语法的原因，起始/停止/步进不足够使用时，使用此标识。
* **"no-cluster"**：该命令不应在Redis Cluster中注册，因为它无法与之配合工作，例如无法报告键的位置、以编程方式创建键名或其他任何原因。
* **"no-auth"**：此命令可以由未经身份验证的客户端运行。通常由用于对客户端进行身份验证的命令使用。
* **"may-replicate"**：即使它不是写命令，该命令可能会产生复制流量。
* **"no-mandatory-keys"**：该命令可以接受的所有键都是可选的。
* **"blocking"**：该命令可能会阻塞客户端。
* **"allow-busy"**：在服务器被脚本或缓慢模块命令阻塞的情况下，允许执行该命令，参见RedisModule_Yield。
* **"getchannels-api"**：该命令实现了返回作为频道的参数的接口。

以下是全部数据，请保留格式：
最后三个参数指定新命令的哪些参数是 Redis 键。更多信息请参考 [https://redis.io/commands/command](https://redis.io/commands/command)。

* `firstkey`: 第一个关键字参数的基于1的索引。
              位置0始终是命令名称本身。
              命令没有关键字参数时为0。
* `lastkey`: 最后一个关键字参数的基于1的索引。
              负数表示从最后一个参数开始反向计数
              (-1表示提供的最后一个参数)
              命令没有关键字参数时为0。
* `keystep`: 第一个关键字参数索引和最后一个关键字参数索引之间的步长。
              命令没有关键字参数时为0。

这些信息被ACL，Cluster和`COMMAND`命令使用。

注意：上述方案仅有限地用于查找存在于固定索引的键。
对于非平凡的键参数，您可以传递0,0,0并使用 [`RedisModule_SetCommandInfo`](#RedisModule_SetCommandInfo) 使用更高级的方案设置键规范，并使用 [`RedisModule_SetCommandACLCategories`](#RedisModule_SetCommandACLCategories) 设置Redis ACL命令的类别。

<span id="RedisModule_GetCommand"></span>

### `RedisModule_GetCommand`


    RedisModuleCommand *RedisModule_GetCommand(RedisModuleCtx *ctx,
                                               const char *name);

**自 7.0.0 版本起可用**

根据命令名称获取一个表示模块命令的不透明结构。
此结构在一些与命令相关的API中使用。

如果发生以下错误，将返回NULL：

* 命令未找到
* 命令不是模块命令
* 命令不属于调用模块

<span id="RedisModule_CreateSubcommand"></span>

### `RedisModule_CreateSubcommand`

    int RedisModule_CreateSubcommand(RedisModuleCommand *parent,
                                     const char *name,
                                     RedisModuleCmdFunc cmdfunc,
                                     const char *strflags,
                                     int firstkey,
                                     int lastkey,
                                     int keystep);

**可用于：**7.0.0

与[`RedisModule_CreateCommand`](#RedisModule_CreateCommand)非常相似，只是它用于创建与另一个容器命令关联的子命令。

例子: 如果一个模块有一个配置命令MODULE.CONFIG, 那么GET和SET应该是单独的子命令, 而MODULE.CONFIG是一个命令, 但不应该注册为一个有效的'funcptr':

     if (RedisModule_CreateCommand(ctx,"module.config",NULL,"",0,0,0) == REDISMODULE_ERR)
         return REDISMODULE_ERR;

     RedisModuleCommand *parent = RedisModule_GetCommand(ctx,,"module.config");

     if (RedisModule_CreateSubcommand(parent,"set",cmd_config_set,"",0,0,0) == REDISMODULE_ERR)
        return REDISMODULE_ERR;

     if (RedisModule_CreateSubcommand(parent,"get",cmd_config_get,"",0,0,0) == REDISMODULE_ERR)
        return REDISMODULE_ERR;

在成功时返回 `REDISMODULE_OK` ，在以下错误情况下返回 `REDISMODULE_ERR` ：

* 解析 `strflags` 时发生错误
* 命令标记为 `no-cluster`，但启用了集群模式
* `parent` 已经是一个子命令（我们不允许多层命令嵌套）
* `parent` 是一个具有实现（`RedisModuleCmdFunc`）的命令（父命令应该是纯粹的子命令容器）
* `parent` 已经有一个名为 `name` 的子命令
* 在 `RedisModule_OnLoad` 外创建子命令

<span id="RedisModule_SetCommandACLCategories"></span>

### `RedisModule_SetCommandACLCategories`
### `RedisModule_SetCommandACLCategories`

    int RedisModule_SetCommandACLCategories(RedisModuleCommand *command,
                                            const char *aclflags);

**可用版本：** 7.2.0

[`RedisModule_SetCommandACLCategories`](#RedisModule_SetCommandACLCategories) 可用于为模块命令和子命令设置 ACL 类别。ACL 类别的集合应传递为以空格分隔的 C 字符串 `aclflags`。

示例，acl标志“write slow”将命令标记为写入和缓慢ACL类别的一部分。

成功返回`REDISMODULE_OK`。错误返回`REDISMODULE_ERR`。

只能在`RedisModule_OnLoad`函数中调用此函数。如果在此函数之外调用，将返回错误。

<span id="RedisModule_SetCommandInfo"></span>

### `RedisModule_SetCommandInfo`


    int RedisModule_SetCommandInfo(RedisModuleCommand *command,
                                   const RedisModuleCommandInfo *info);

**自 7.0.0 版本起可用**

设置额外的命令信息。

该文本影响`COMMAND`、`COMMAND INFO`和`COMMAND DOCS`的输出，集群、ACL，并用于在调用到模块代码之前过滤参数错误的命令。

这个函数可以在使用[`RedisModule_CreateCommand`](#RedisModule_CreateCommand)创建命令并使用[`RedisModule_GetCommand`](#RedisModule_GetCommand)获取命令指针后调用。该信息对于每个命令只能设置一次，并具有以下结构：

    typedef struct RedisModuleCommandInfo {
        const RedisModuleCommandInfoVersion *version;
        const char *summary;
        const char *complexity;
        const char *since;
        RedisModuleCommandHistoryEntry *history;
        const char *tips;
        int arity;
        RedisModuleCommandKeySpec *key_specs;
        RedisModuleCommandArg *args;
    } RedisModuleCommandInfo;

除了 `version` 字段之外，所有字段都是可选的。字段的解释:

- `version`：这个字段可以与不同版本的Redis兼容。
  始终将这个字段设置为`REDISMODULE_COMMAND_INFO_VERSION`。

- `summary`: 命令的简短描述（可选）。

- `complexity`: 复杂性描述（可选）。

- `since`: 命令被引入的版本（可选）。
  注意：指定的版本应该是模块的版本，而不是 Redis 的版本。

- `history`: 一个 `RedisModuleCommandHistoryEntry` 数组（可选），包含以下字段的结构体：

        const char *since;
        const char *changes;

    `since` is a version string and `changes` is a string describing the
    changes. The array is terminated by a zeroed entry, i.e. an entry with
    both strings set to NULL.

- `tips`：一个以空格分隔的字符串，包含与此命令相关的提示，用于客户端和代理。请参阅[https://redis.io/topics/command-tips](https://redis.io/topics/command-tips)。

- `arity`: 参数个数，包括命令名称本身。正数指定准确的参数个数，负数指定最小参数个数，因此可以使用-N来表示 >= N 的情况。Redis 在将调用传递给模块之前会对其进行验证，因此这可以替代模块命令实现中的参数个数检查。如果命令有子命令，则含有0（或省略 arity 字段）的值相当于 -2，否则相当于 -1。

- `key_specs`：一个由 `RedisModuleCommandKeySpec` 组成的数组，以零填充结尾。这是一种尝试更好描述键参数位置的方案，比旧的 [`RedisModule_CreateCommand`](#RedisModule_CreateCommand) 的 `firstkey`、`lastkey`、`keystep` 参数更需要，如果仅凭这三个参数无法描述键的位置。有两个步骤来获取键的位置：*开始搜索* (BS)，在哪个索引上应找到第一个键，和 *查找键* (FK)，相对于 BS 的输出，描述了哪些参数是键。此外，还有特定于键的标志。

    Key-specs cause the triplet (firstkey, lastkey, keystep) given in
    RedisModule_CreateCommand to be recomputed, but it is still useful to provide
    these three parameters in RedisModule_CreateCommand, to better support old Redis
    versions where RedisModule_SetCommandInfo is not available.

    Note that key-specs don't fully replace the "getkeys-api" (see
    RedisModule_CreateCommand, RedisModule_IsKeysPositionRequest and RedisModule_KeyAtPosWithFlags) so
    it may be a good idea to supply both key-specs and implement the
    getkeys-api.

    A key-spec has the following structure:

        typedef struct RedisModuleCommandKeySpec {
            const char *notes;
            uint64_t flags;
            RedisModuleKeySpecBeginSearchType begin_search_type;
            union {
                struct {
                    int pos;
                } index;
                struct {
                    const char *keyword;
                    int startfrom;
                } keyword;
            } bs;
            RedisModuleKeySpecFindKeysType find_keys_type;
            union {
                struct {
                    int lastkey;
                    int keystep;
                    int limit;
                } range;
                struct {
                    int keynumidx;
                    int firstkey;
                    int keystep;
                } keynum;
            } fk;
        } RedisModuleCommandKeySpec;

    Explanation of the fields of RedisModuleCommandKeySpec:

    * `notes`: Optional notes or clarifications about this key spec.

    * `flags`: A bitwise or of key-spec flags described below.

    * `begin_search_type`: This describes how the first key is discovered.
      There are two ways to determine the first key:

        * `REDISMODULE_KSPEC_BS_UNKNOWN`: There is no way to tell where the
          key args start.
        * `REDISMODULE_KSPEC_BS_INDEX`: Key args start at a constant index.
        * `REDISMODULE_KSPEC_BS_KEYWORD`: Key args start just after a
          specific keyword.

    * `bs`: This is a union in which the `index` or `keyword` branch is used
      depending on the value of the `begin_search_type` field.

        * `bs.index.pos`: The index from which we start the search for keys.
          (`REDISMODULE_KSPEC_BS_INDEX` only.)

        * `bs.keyword.keyword`: The keyword (string) that indicates the
          beginning of key arguments. (`REDISMODULE_KSPEC_BS_KEYWORD` only.)

        * `bs.keyword.startfrom`: An index in argv from which to start
          searching. Can be negative, which means start search from the end,
          in reverse. Example: -2 means to start in reverse from the
          penultimate argument. (`REDISMODULE_KSPEC_BS_KEYWORD` only.)

    * `find_keys_type`: After the "begin search", this describes which
      arguments are keys. The strategies are:

        * `REDISMODULE_KSPEC_BS_UNKNOWN`: There is no way to tell where the
          key args are located.
        * `REDISMODULE_KSPEC_FK_RANGE`: Keys end at a specific index (or
          relative to the last argument).
        * `REDISMODULE_KSPEC_FK_KEYNUM`: There's an argument that contains
          the number of key args somewhere before the keys themselves.

      `find_keys_type` and `fk` can be omitted if this keyspec describes
      exactly one key.

    * `fk`: This is a union in which the `range` or `keynum` branch is used
      depending on the value of the `find_keys_type` field.

        * `fk.range` (for `REDISMODULE_KSPEC_FK_RANGE`): A struct with the
          following fields:

            * `lastkey`: Index of the last key relative to the result of the
              begin search step. Can be negative, in which case it's not
              relative. -1 indicates the last argument, -2 one before the
              last and so on.

            * `keystep`: How many arguments should we skip after finding a
              key, in order to find the next one?

            * `limit`: If `lastkey` is -1, we use `limit` to stop the search
              by a factor. 0 and 1 mean no limit. 2 means 1/2 of the
              remaining args, 3 means 1/3, and so on.

        * `fk.keynum` (for `REDISMODULE_KSPEC_FK_KEYNUM`): A struct with the
          following fields:

            * `keynumidx`: Index of the argument containing the number of
              keys to come, relative to the result of the begin search step.

            * `firstkey`: Index of the fist key relative to the result of the
              begin search step. (Usually it's just after `keynumidx`, in
              which case it should be set to `keynumidx + 1`.)

            * `keystep`: How many arguments should we skip after finding a
              key, in order to find the next one?

    Key-spec flags:

    The first four refer to what the command actually does with the *value or
    metadata of the key*, and not necessarily the user data or how it affects
    it. Each key-spec may must have exactly one of these. Any operation
    that's not distinctly deletion, overwrite or read-only would be marked as
    RW.

    * `REDISMODULE_CMD_KEY_RO`: Read-Only. Reads the value of the key, but
      doesn't necessarily return it.

    * `REDISMODULE_CMD_KEY_RW`: Read-Write. Modifies the data stored in the
      value of the key or its metadata.

    * `REDISMODULE_CMD_KEY_OW`: Overwrite. Overwrites the data stored in the
      value of the key.

    * `REDISMODULE_CMD_KEY_RM`: Deletes the key.

    The next four refer to *user data inside the value of the key*, not the
    metadata like LRU, type, cardinality. It refers to the logical operation
    on the user's data (actual input strings or TTL), being
    used/returned/copied/changed. It doesn't refer to modification or
    returning of metadata (like type, count, presence of data). ACCESS can be
    combined with one of the write operations INSERT, DELETE or UPDATE. Any
    write that's not an INSERT or a DELETE would be UPDATE.

    * `REDISMODULE_CMD_KEY_ACCESS`: Returns, copies or uses the user data
      from the value of the key.

    * `REDISMODULE_CMD_KEY_UPDATE`: Updates data to the value, new value may
      depend on the old value.

    * `REDISMODULE_CMD_KEY_INSERT`: Adds data to the value with no chance of
      modification or deletion of existing data.

    * `REDISMODULE_CMD_KEY_DELETE`: Explicitly deletes some content from the
      value of the key.

    Other flags:

    * `REDISMODULE_CMD_KEY_NOT_KEY`: The key is not actually a key, but 
      should be routed in cluster mode as if it was a key.

    * `REDISMODULE_CMD_KEY_INCOMPLETE`: The keyspec might not point out all
      the keys it should cover.

    * `REDISMODULE_CMD_KEY_VARIABLE_FLAGS`: Some keys might have different
      flags depending on arguments.

- `args`：一个由`RedisModuleCommandArg`数组组成的，以将最后一个元素memset为零为结束标志的数组。`RedisModuleCommandArg`是一个具有以下所述字段的结构。

        typedef struct RedisModuleCommandArg {
            const char *name;
            RedisModuleCommandArgType type;
            int key_spec_index;
            const char *token;
            const char *summary;
            const char *since;
            int flags;
            struct RedisModuleCommandArg *subargs;
        } RedisModuleCommandArg;

    Explanation of the fields:

    * `name`: Name of the argument.

    * `type`: The type of the argument. See below for details. The types
      `REDISMODULE_ARG_TYPE_ONEOF` and `REDISMODULE_ARG_TYPE_BLOCK` require
      an argument to have sub-arguments, i.e. `subargs`.

    * `key_spec_index`: If the `type` is `REDISMODULE_ARG_TYPE_KEY` you must
      provide the index of the key-spec associated with this argument. See
      `key_specs` above. If the argument is not a key, you may specify -1.

    * `token`: The token preceding the argument (optional). Example: the
      argument `seconds` in `SET` has a token `EX`. If the argument consists
      of only a token (for example `NX` in `SET`) the type should be
      `REDISMODULE_ARG_TYPE_PURE_TOKEN` and `value` should be NULL.

    * `summary`: A short description of the argument (optional).

    * `since`: The first version which included this argument (optional).

    * `flags`: A bitwise or of the macros `REDISMODULE_CMD_ARG_*`. See below.

    * `value`: The display-value of the argument. This string is what should
      be displayed when creating the command syntax from the output of
      `COMMAND`. If `token` is not NULL, it should also be displayed.

    Explanation of `RedisModuleCommandArgType`:

    * `REDISMODULE_ARG_TYPE_STRING`: String argument.
    * `REDISMODULE_ARG_TYPE_INTEGER`: Integer argument.
    * `REDISMODULE_ARG_TYPE_DOUBLE`: Double-precision float argument.
    * `REDISMODULE_ARG_TYPE_KEY`: String argument representing a keyname.
    * `REDISMODULE_ARG_TYPE_PATTERN`: String, but regex pattern.
    * `REDISMODULE_ARG_TYPE_UNIX_TIME`: Integer, but Unix timestamp.
    * `REDISMODULE_ARG_TYPE_PURE_TOKEN`: Argument doesn't have a placeholder.
      It's just a token without a value. Example: the `KEEPTTL` option of the
      `SET` command.
    * `REDISMODULE_ARG_TYPE_ONEOF`: Used when the user can choose only one of
      a few sub-arguments. Requires `subargs`. Example: the `NX` and `XX`
      options of `SET`.
    * `REDISMODULE_ARG_TYPE_BLOCK`: Used when one wants to group together
      several sub-arguments, usually to apply something on all of them, like
      making the entire group "optional". Requires `subargs`. Example: the
      `LIMIT offset count` parameters in `ZRANGE`.

    Explanation of the command argument flags:

    * `REDISMODULE_CMD_ARG_OPTIONAL`: The argument is optional (like GET in
      the SET command).
    * `REDISMODULE_CMD_ARG_MULTIPLE`: The argument may repeat itself (like
      key in DEL).
    * `REDISMODULE_CMD_ARG_MULTIPLE_TOKEN`: The argument may repeat itself,
      and so does its token (like `GET pattern` in SORT).

成功时返回`REDISMODULE_OK`。出错时返回`REDISMODULE_ERR`，并将`errno`设置为EINVAL（如果提供了无效的信息）或EEXIST（如果信息已经被设置）。如果信息无效，则记录一个警告，解释哪部分信息无效以及原因。

<span id= "section-module-information-and-time-measurement"></span>

## 模块信息和时间测量

<span id="RedisModule_IsModuleNameBusy"></span>

### `RedisModule_IsModuleNameBusy`

### `RedisModule_IsModuleNameBusy`函数

    int RedisModule_IsModuleNameBusy(const char *name);

**可用自：** 4.0.3

如果模块名称已被使用，则返回非零。
否则返回零。

<span id="RedisModule_Milliseconds"></span>

### `RedisModule_Milliseconds`

### `RedisModule_Milliseconds`

    mstime_t RedisModule_Milliseconds(void);

**可用于版本:** 4.0.0

返回当前的UNIX时间（以毫秒为单位）。

<span id="RedisModule_MonotonicMicroseconds"></span>

### `RedisModule_MonotonicMicroseconds`

    uint64_t RedisModule_MonotonicMicroseconds(void);

**可用版本：** 7.0.0

以微秒为单位返回相对于任意时间点的计数器。

<span id="RedisModule_毫秒"></span>

### `RedisModule_Microseconds`

RedisModule_Microseconds函数用于返回以微秒为单位的当前时间。它没有参数。函数返回一个64位有符号整数，表示自Epoch（1970年1月1日）以来经过的微秒数。

    ustime_t RedisModule_Microseconds(void);

**可用自：**7.2.0

返回当前的UNIX时间，以微秒为单位

<span id="RedisModule_CachedMicroseconds"></span>

### `RedisModule_CachedMicroseconds`

### `RedisModule_CachedMicroseconds`

    ustime_t RedisModule_CachedMicroseconds(void);

**从版本7.2.0开始可用**

以毫秒为单位返回缓存的UNIX时间。
它在服务器cron任务和执行命令之前进行更新。
对于复杂的调用堆栈非常有用，例如导致键空间通知的命令，
导致模块执行[`RedisModule_Call`](# RedisModule_Call)，
引起另一个通知等等。
确保所有这些回调使用相同的时钟是有意义的。

<span id="RedisModule_BlockedClientMeasureTimeStart"></span>

### `RedisModule_BlockedClientMeasureTimeStart`
### `RedisModule_BlockedClientMeasureTimeStart`
### `RedisModule_BlockedClientMeasureTimeStart`

    int RedisModule_BlockedClientMeasureTimeStart(RedisModuleBlockedClient *bc);

**可用自：**6.2.0

在调用 [`RedisModule_BlockedClientMeasureTimeEnd()`](#RedisModule_BlockedClientMeasureTimeEnd) 时，将一个时间点标记为用于计算经过的执行时间的起始时间。

在同一个命令中，可以多次调用 [`RedisModule_BlockedClientMeasureTimeStart()`](#RedisModule_BlocekedClientMeasureTimeStart) 和 [`RedisModule_BlockedClientMeasureTimeEnd()`](#RedisModule_BlocekedClientMeasureTimeEnd) 以累积独立的时间间隔到后台持续时间中。

此方法始终返回 `REDISMODULE_OK`。

<span id="RedisModule_BlockedClientMeasureTimeEnd"></span>

### `RedisModule_BlockedClientMeasureTimeEnd`

### `RedisModule_BlockedClientMeasureTimeEnd`

    int RedisModule_BlockedClientMeasureTimeEnd(RedisModuleBlockedClient *bc);

**可用版本:** 6.2.0

在计算经过的执行时间时，标记一个将用作结束时间的时间点。
成功时返回`REDISMODULE_OK`。
只有在之前未定义开始时间（即未调用[`RedisModule_BlockedClientMeasureTimeStart`](#RedisModule_BlockedClientMeasureTimeStart)）时，此方法才会返回`REDISMODULE_ERR`。

<span id="RedisModule_Yield"></span>

### `RedisModule_Yield`

    void RedisModule_Yield(RedisModuleCtx *ctx, int flags, const char *busy_reply);

**可用于版本：**7.0.0

此 API 允许模块让 Redis 处理后台任务，并在模块命令的长时间阻塞执行期间执行一些命令。
模块可以定期调用此 API。
flags 是这些的位掩码：

- `REDISMODULE_YIELD_FLAG_NONE`: 没有特殊标志，可以执行一些后台操作，但不能处理客户端命令。
- `REDISMODULE_YIELD_FLAG_CLIENTS`: Redis也可以处理客户端命令。

`busy_reply`参数是可选的，可以用来控制在`-BUSY`错误代码后的冗长错误字符串。

当使用`REDISMODULE_YIELD_FLAG_CLIENTS`时，Redis将只在`busy-reply-threshold`配置所定义的时间后开始处理客户端命令，在此情况下，Redis将开始拒绝大多数命令，并显示`-BUSY`错误，但允许标记为`allow-busy`的命令被执行。

此API还可在线程安全的上下文（在锁定时）和加载期间（在`rdb_load`回调中）使用，在加载期间，它将拒绝带有`-LOADING`错误的命令。

<span id="RedisModule_SetModuleOptions"></span>

### `RedisModule_SetModuleOptions`


    void RedisModule_SetModuleOptions(RedisModuleCtx *ctx, int options);

**可用版本**：6.0.0

使用标志设置定义能力或行为位标志。

`REDISMODULE_OPTIONS_HANDLE_IO_ERRORS`：
一般情况下，模块不需要关心这个选项，因为如果发生读取错误，进程将会立即终止。但是，设置这个标志将允许在启用时使用repl-diskless-load。
模块在读取之后，在使用已读取的数据之前应该使用[`RedisModule_IsIOError`](#RedisModule_IsIOError)来检查错误，并在出现错误时向上传播错误，并能够释放部分填充的值和所有相关的分配。

`REDISMODULE_OPTION_NO_IMPLICIT_SIGNAL_MODIFIED`:
参见[`RedisModule_SignalModifiedKey()`](#RedisModule_SignalModifiedKey)。

`REDISMODULE_OPTIONS_HANDLE_REPL_ASYNC_LOAD`：
设置此标志表示模块对磁盘无关的异步复制（repl-diskless-load=swapdb）有意识，并且在复制期间，Redis 可能会提供读取而不是阻塞并显示 LOADING 状态。

`REDISMODULE_OPTIONS_ALLOW_NESTED_KEYSPACE_NOTIFICATIONS`：
声明模块希望获取嵌套键空间通知。
默认情况下，Redis不会触发在键空间通知回调中发生的键空间通知。此标志允许更改此行为并触发嵌套键空间通知。注意：如果启用，则模块应保护自己免受无限递归的影响。

<span id="RedisModule_SignalModifiedKey"></span>

### `RedisModule_SignalModifiedKey`
### `RedisModule_SignalModifiedKey`

    int RedisModule_SignalModifiedKey(RedisModuleCtx *ctx,
                                      RedisModuleString *keyname);

**可用自：** 6.0.0

表明密钥已从用户的角度进行了修改（即使无效的 WATCH 和客户端缓存）。

当写入键关闭时，除非使用`RedisModule_SetModuleOptions()`设置了选项`REDISMODULE_OPTION_NO_IMPLICIT_SIGNAL_MODIFIED`，否则会自动执行此操作。

<span id="模块的自动内存管理"></span>

## 自动模块内存管理

<span id="RedisModule_AutoMemory"></span>

### `RedisModule_AutoMemory`

    void RedisModule_AutoMemory(RedisModuleCtx *ctx);

**可用自：**4.0.0

启用自动内存管理。

函数必须作为命令实现的第一个函数调用，该命令实现希望使用自动内存。

当启用时，自动内存管理会跟踪并在命令返回后自动释放键、调用回复和 Redis 字符串对象。在大多数情况下，这消除了调用以下函数的需要：

1. [`RedisModule_CloseKey()`](#RedisModule_CloseKey)
2. [`RedisModule_FreeCallReply()`](#RedisModule_FreeCallReply)
3. [`RedisModule_FreeString()`](#RedisModule_FreeString)

这些函数在启用自动内存管理的情况下仍然可以使用，以优化例如进行大量分配的循环。

<span id="section-string-objects-apis"></span>

## 字符串对象API

<span id="RedisModule_CreateString"></span>

### `RedisModule_CreateString`

### `RedisModule_CreateString`

    RedisModuleString *RedisModule_CreateString(RedisModuleCtx *ctx,
                                                const char *ptr,
                                                size_t len);

**可用于版本：** 4.0.0

创建一个新的模块字符串对象。返回的字符串必须使用[`RedisModule_FreeString()`](#RedisModule_FreeString)进行释放，除非启用了自动内存管理。

字符串是通过复制从`ptr`开始的`len`字节创建的。没有保留对传入缓冲区的引用。

模块上下文“ctx”是可选的，如果您想要在上下文范围之外创建一个字符串，它可能为NULL。但在这种情况下，自动内存管理将不可用，字符串内存必须手动管理。

<span id="RedisModule_CreateStringPrintf"></span>

### `RedisModule_CreateStringPrintf`

### `RedisModule_CreateStringPrintf`

    RedisModuleString *RedisModule_CreateStringPrintf(RedisModuleCtx *ctx,
                                                      const char *fmt,
                                                      ...);

**可用自：** 4.0.0

从printf格式和参数创建一个新的模块字符串对象。
除非启用了自动内存分配，否则必须使用[`RedisModule_FreeString()`](#RedisModule_FreeString)释放返回的字符串。

该文本是使用sds格式化函数`sdscatvprintf()`创建的字符串。

如果有必要，传递的上下文 'ctx' 可能为 NULL，请参阅 [`RedisModule_CreateString()`](#RedisModule_CreateString) 文档获取更多信息。

<span id="RedisModule_CreateStringFromLongLong"></span>


### `RedisModule_CreateStringFromLongLong`

`RedisModule_CreateStringFromLongLong`函数

    RedisModuleString *RedisModule_CreateStringFromLongLong(RedisModuleCtx *ctx,
                                                            long long ll);

**可用自：**4.0.0

类似于[`RedisModule_CreateString()`](#RedisModule_CreateString)，但是从`long long`整数开始创建字符串，而不是传入缓冲区和长度。

返回的字符串必须使用[`RedisModule_FreeString()`](#RedisModule_FreeString)释放，或者启用自动内存管理。

如果需要，传递的上下文'ctx'可以为NULL，请参阅[`RedisModule_CreateString()`](#RedisModule_CreateString) 的文档以获取更多信息。

<span id="RedisModule_CreateStringFromULongLong"></span>

### `RedisModule_CreateStringFromULongLong`

    RedisModuleString *RedisModule_CreateStringFromULongLong(RedisModuleCtx *ctx,
                                                             unsigned long long ull);

**可用自:** 7.0.3

与 [`RedisModule_CreateString()`](#RedisModule_CreateString) 类似，但是根据一个 `unsigned long long` 整数创建一个字符串，而不是使用缓冲区和长度。

返回的字符串必须通过[`RedisModule_FreeString()`](#RedisModule_FreeString)释放，或者启用自动内存管理。

如果需要，传递的上下文 'ctx' 可能为NULL，请参阅 [`RedisModule_CreateString()`](#RedisModule_CreateString) 文档以获取更多信息。

`<span id="RedisModule_CreateStringFromDouble"></span>`

### `RedisModule_CreateStringFromDouble`


    RedisModuleString *RedisModule_CreateStringFromDouble(RedisModuleCtx *ctx,
                                                          double d);

**可用于：** 6.0.0

类似于 [`RedisModule_CreateString()`](#RedisModule_CreateString)，但是从一个双精度浮点数创建一个字符串，而不是使用缓冲区和长度。

返回的字符串必须使用[`RedisModule_FreeString()`](#RedisModule_FreeString)释放，或通过启用自动内存管理释放。

<span id="RedisModule_CreateStringFromLongDouble"></span>

### `RedisModule_CreateStringFromLongDouble`

    RedisModuleString *RedisModule_CreateStringFromLongDouble(RedisModuleCtx *ctx,
                                                              long double ld,
                                                              int humanfriendly);

**可用自：** 6.0.0

类似于 [`RedisModule_CreateString()`](#RedisModule_CreateString)，但是从一个 `long double` 开始创建一个字符串。

返回的字符串必须使用 [`RedisModule_FreeString()`](#RedisModule_FreeString) 释放，或者启用自动内存管理。

如果需要，传递的上下文'ctx'可能为NULL，请参见[`RedisModule_CreateString()`](#RedisModule_CreateString)的文档了解更多信息。

<span id="RedisModule_CreateStringFromString"></span>

### `RedisModule_CreateStringFromString`

```
函数原型：RedisModuleString *RedisModule_CreateStringFromString(RedisModuleCtx *ctx, const RedisModuleString *str)；

函数功能：以一个已存在的字符串对象.RedisModuleString为基础创建一个新的字符串对象.。因此新对象与参数字符串对象共享底层数据.因此当参数被回收时，这个新创建的字符串对象也不能再被使用，而且任何对参数字符串对象进行修改的操作都可能导致这个新字符串对象所得到数据完全改变。

函数返回值：新创建的字符串对象.
```


    RedisModuleString *RedisModule_CreateStringFromString(RedisModuleCtx *ctx,
                                                          const RedisModuleString *str);

**可用自：**4.0.0

与[`RedisModule_CreateString()`](#RedisModule_CreateString)相似，但是可以从另一个`RedisModuleString`创建一个字符串。

返回的字符串必须使用 [`RedisModule_FreeString()`](#RedisModule_FreeString) 释放，或者启用自动内存管理。

如果需要，传递的上下文'ctx'可以为NULL，请参阅[`RedisModule_CreateString()`](#RedisModule_CreateString)文档获取更多信息。

<span id="RedisModule_CreateStringFromStreamID"></span>

### `RedisModule_CreateStringFromStreamID`

### `RedisModule_CreateStringFromStreamID`

    RedisModuleString *RedisModule_CreateStringFromStreamID(RedisModuleCtx *ctx,
                                                            const RedisModuleStreamID *id);

**可用版本：**6.2.0

从流ID创建一个字符串。返回的字符串必须使用[`RedisModule_FreeString()`](#RedisModule_FreeString)释放，除非启用了自动内存。

如果需要，传递的上下文`ctx`可能是空值。有关更多信息，请参阅[`RedisModule_CreateString()`](#RedisModule_CreateString) 文档。

<span id="RedisModule_FreeString"></span>

### `RedisModule_FreeString`

RedisModule_FreeString释放由RedisModule_CreateString函数分配的字符串对象，并将其内存释放回Redis内存管理器。

    void RedisModule_FreeString(RedisModuleCtx *ctx, RedisModuleString *str);

**可用自：**4.0.0

使用返回新字符串对象的Redis模块API调用中获得的模块字符串对象，释放它。

在启用自动内存管理的情况下，仍可以调用此函数。在这种情况下，字符串将立即释放，并从要在最后释放的字符串池中删除。

如果使用NULL上下文'ctx'创建了字符串，则在释放字符串时也可以将ctx传递为NULL（但传递上下文不会产生任何问题）。使用上下文创建的字符串应该在释放时也传递上下文，所以如果您想稍后在上下文之外释放字符串，请确保使用NULL上下文来创建它。

<span id="RedisModule_RetainString"></span>

### `RedisModule_RetainString`

### `RedisModule_RetainString`（保留字符串）

    void RedisModule_RetainString(RedisModuleCtx *ctx, RedisModuleString *str);

**可用于：** 4.0.0

对该函数的每次调用都会使字符串'str'需要额外调用[`RedisModule_FreeString()`](#RedisModule_FreeString)来真正释放字符串。请注意，以启用模块自动内存管理得到的字符串的自动释放当作一个[`RedisModule_FreeString()`](#RedisModule_FreeString)调用（它只是自动执行）。

通常情况下，当同时满足以下条件时，您希望调用此函数：

1. 您已启用自动内存管理。
2. 您想创建字符串对象。
3. 您创建的这些字符串对象需要在回调函数（例如命令实现）创建并返回后才能继续存在。

通常情况下，您希望将创建的字符串对象存储到自己的数据结构中，例如在实现新的数据类型时。

请注意，当关闭内存管理时，您不需要调用RetainString()，因为创建字符串将始终导致在回调函数返回后仍然存在的字符串，如果没有执行FreeString()调用。

可以使用NULL上下文调用此函数。

当字符串要保留较长时间时，最好还调用[`RedisModule_TrimStringAllocation()`](#RedisModule_TrimStringAllocation)来优化内存使用。

Threaded指向其他线程保留的字符串的模块*必须*在保留字符串后立即显式地修剪分配。不这样做可能导致自动修剪，这是不安全的多线程操作。

<span id="RedisModule_HoldString"></span>

### `RedisModule_HoldString`
`RedisModule_HoldString`函数用于增加一个字符串对象的引用计数。

    RedisModuleString* RedisModule_HoldString(RedisModuleCtx *ctx,
                                              RedisModuleString *str);

**自从可用：**6.0.7


这个函数可以用来替代 `RedisModule_RetainString()`。
两者之间的主要区别在于，这个函数永远都会成功，而 `RedisModule_RetainString()` 可能会因为断言失败而失败。

函数返回一个指向`RedisModuleString`的指针，该指针由调用者拥有。在上下文禁用自动内存管理时，需要调用[`RedisModule_FreeString()`](#RedisModule_FreeString)来释放字符串。当启用自动内存管理时，可以选择调用[`RedisModule_FreeString()`](#RedisModule_FreeString)来释放字符串，或者由自动化机制来释放。

这个函数比[`RedisModule_CreateStringFromString()`](#RedisModule_CreateStringFromString)更高效，因为在可能的情况下，它避免复制底层的`RedisModuleString`。使用这个函数的缺点是可能无法对返回的`RedisModuleString`使用[`RedisModule_StringAppendBuffer()`](#RedisModule_StringAppendBuffer)。

有可能用带有NULL上下文的方式调用此函数。

当字符串将被长时间保留时，最好还调用 [`RedisModule_TrimStringAllocation()`](# RedisModule_TrimStringAllocation) 来优化内存使用。

线程化模块必须明确地修剪分配，以便尽快将字符串保持在其它线程中。不这样做可能导致自动修剪不是线程安全的情况。

<span id="RedisModule_StringPtrLen"></span>

### `RedisModule_StringPtrLen`


    const char *RedisModule_StringPtrLen(const RedisModuleString *str,
                                         size_t *len);

**可用自：**4.0.0

给定一个字符串模块对象，该函数返回字符串指针和字符串的长度。返回的指针和长度只能用于只读访问，不能修改。

<span id="RedisModule_StringToLongLong"></span>

### `RedisModule_StringToLongLong`

### `RedisModule_StringToLongLong`

    int RedisModule_StringToLongLong(const RedisModuleString *str, long long *ll);

**可用自：**4.0.0

将字符串转换为 `long long` 整数，并将其存储在 `*ll` 中。
成功时返回 `REDISMODULE_OK`。如果字符串无法解析为有效的、严格的 `long long`（没有前后空格），则返回 `REDISMODULE_ERR`。

<span id="RedisModule_StringToULongLong"></span>

### `RedisModule_StringToULongLong`

    int RedisModule_StringToULongLong(const RedisModuleString *str,
                                      unsigned long long *ull);

**可用自：**7.0.3

将字符串转换为`unsigned long long`整数，并将其存储在`*ull`中。
成功时返回`REDISMODULE_OK`。如果字符串无法解析为有效的、严格的`unsigned long long`（前后没有空格），则返回`REDISMODULE_ERR`。

<span id="RedisModule_StringToDouble"></span>


### `RedisModule_StringToDouble`

`RedisModule_StringToDouble` 函数

    int RedisModule_StringToDouble(const RedisModuleString *str, double *d);

**可用于版本：**4.0.0

将字符串转换为双精度浮点数，并将其存储在 `*d` 中。
成功时返回 `REDISMODULE_OK`，如果字符串不是有效的双精度浮点数表示，则返回 `REDISMODULE_ERR`。

<span id="RedisModule_StringToLongDouble"></span>将字符串转换为`long double`类型的函数

### `RedisModule_StringToLongDouble`

### `RedisModule_StringToLongDouble`

    int RedisModule_StringToLongDouble(const RedisModuleString *str,
                                       long double *ld);

**可用自：**6.0.0

将字符串转换为长双精度浮点数，并将其存储在`*ld`中。
如果字符串不是有效的双精度值的字符串表示，则返回`REDISMODULE_OK`，否则返回`REDISMODULE_ERR`。

<span id="RedisModule_StringToStreamID"></span>


### `RedisModule_StringToStreamID`

### `RedisModule_StringToStreamID`

    int RedisModule_StringToStreamID(const RedisModuleString *str,
                                     RedisModuleStreamID *id);

**可用自：**6.2.0

将字符串转换为流ID，并将其存储在`*id`中。
成功返回`REDISMODULE_OK`，如果字符串不是有效的流ID字符串表示形式，则返回`REDISMODULE_ERR`。特殊的ID"+"和"-"是允许的。

<span id="RedisModule_StringCompare"></span>

### `RedisModule_StringCompare`


    int RedisModule_StringCompare(const RedisModuleString *a,
                                  const RedisModuleString *b);

**起始版本：** 4.0.0

比较两个字符串对象，如果 a < b，则返回 -1；如果 a == b，则返回 0；如果 a > b，则返回 1。字符串按字节逐个比较，没有任何编码注意或排序尝试。

<span id="RedisModule_StringAppendBuffer"></span>

### `RedisModule_StringAppendBuffer`

    int RedisModule_StringAppendBuffer(RedisModuleCtx *ctx,
                                       RedisModuleString *str,
                                       const char *buf,
                                       size_t len);

**可用自：** 4.0.0

将指定的缓冲区追加到字符串 "str"。该字符串必须是用户创建的、仅被引用一次的字符串，否则将返回`REDISMODULE_ERR`并且不执行操作。

<span id="RedisModule_TrimStringAllocation"></span>

### `RedisModule_TrimStringAllocation`
### `RedisModule_TrimStringAllocation`

    void RedisModule_TrimStringAllocation(RedisModuleString *str);

**可用自：**7.0.0

修剪为`RedisModuleString`分配的可能多余的内存。

有时候一个 `RedisModuleString` 分配的内存可能比需要的更多，通常是通过网络缓冲区构建的 argv 参数造成的。这个函数通过重新分配它们的内存来优化这些字符串，对于那些不是短暂存在而是长时间保留的字符串非常有用。

这个操作*不是线程安全*的，只有在确保对字符串的并发访问时才能调用它。在模块命令的argv字符串中使用它之前，确保字符串对其他线程是可用的，一般是安全的。

目前，当模块命令返回时，Redis 可能还会自动删除保留的字符串。然而，明确地进行此操作仍应该是首选选项：

1. Redis的未来版本可能会放弃自动修剪。
2. 目前实现的自动修剪不是*线程安全的*。
   一个操作最近被保留的字符串的后台线程可能会与自动修剪发生竞争条件，导致数据损坏。

<span id="section-reply-apis"></span>

## 回复接口

这些函数用于向客户端发送回复。

大多数函数通常返回`REDISMODULE_OK`，因此您可以使用`return`以便从命令实现中返回：

    if (... some condition ...)
        return RedisModule_ReplyWithLongLong(ctx,mycount);

### 回复集合函数

启动集合回复后，模块必须调用其他`ReplyWith*`风格的函数以发射集合的元素。
集合类型包括：数组、映射、集合和属性。

当处理事先不知道具体元素数量的集合时，可以使用特殊标志`REDISMODULE_POSTPONED_LEN`（以前是`REDISMODULE_POSTPONED_ARRAY_LEN`）调用函数，并且可以在之后使用`RedisModule_ReplySetLength()`调用来设置实际元素数量（如果有多个，则设置最新的“开放”计数）。

<span id="RedisModule_WrongArity"></span>

### `RedisModule_WrongArity`

    int RedisModule_WrongArity(RedisModuleCtx *ctx);

**可用版本:** 4.0.0

发送关于给定命令参数数量的错误，
在错误消息中引用命令名称。返回`REDISMODULE_OK`。

示例：

    if (argc != 3) return RedisModule_WrongArity(ctx);

<span id="RedisModule_ReplyWithLongLong"></span>

### `RedisModule_ReplyWithLongLong`

### `RedisModule_ReplyWithLongLong`

    int RedisModule_ReplyWithLongLong(RedisModuleCtx *ctx, long long ll);

**可用自：**4.0.0

以指定的 `long long` 值向客户端发送一个整数回复。
该函数始终返回 `REDISMODULE_OK`。

<span id="RedisModule_ReplyWithError"></span>
用错导致的红米-ReplyWithError</span>

### `RedisModule_ReplyWithError`

    int RedisModule_ReplyWithError(RedisModuleCtx *ctx, const char *err);

**可用于版本：**4.0.0

以错误信息'err'回复。

注意 'err' 必须包含所有的错误，包括初始错误码。函数仅提供初始的 "-"，所以用法是，例如：

    RedisModule_ReplyWithError(ctx,"ERR Wrong Type");

不仅如此：

    RedisModule_ReplyWithError(ctx,"Wrong Type");

该函数总是返回 `REDISMODULE_OK`。

<span id="RedisModule_ReplyWithErrorFormat"></span>

### `RedisModule_ReplyWithErrorFormat`


    int RedisModule_ReplyWithErrorFormat(RedisModuleCtx *ctx,
                                         const char *fmt,
                                         ...);

**可用自:** 7.2.0

以 printf 格式和参数创建错误回复。

请注意，'fmt'必须包含所有的错误信息，包括初始错误代码。该函数仅提供初始的"-"，因此使用方法如下所示：

    RedisModule_ReplyWithErrorFormat(ctx,"ERR Wrong Type: %s",type);

并不只是:

    RedisModule_ReplyWithErrorFormat(ctx,"Wrong Type: %s",type);

该函数始终返回`REDISMODULE_OK`。

<span id="RedisModule_ReplyWithSimpleString"></span>

### `RedisModule_ReplyWithSimpleString`

### `RedisModule_ReplyWithSimpleString`

    int RedisModule_ReplyWithSimpleString(RedisModuleCtx *ctx, const char *msg);

**可用版本：**4.0.0

回复一个简单的字符串（在RESP协议中为`+... \r\n`）。这些回复只适用于发送小型非二进制字符串且开销较小的情况，如"OK"或类似的回复。

该函数始终返回 `REDISMODULE_OK`。

<span id="RedisModule_ReplyWithArray"></span>


### `RedisModule_ReplyWithArray`

### `RedisModule_ReplyWithArray`

    int RedisModule_ReplyWithArray(RedisModuleCtx *ctx, long len);

支持情况：4.0.0及以上版本

回复一个包含 'len' 个元素的数组类型。

在开始数组回复之后，模块必须根据数组的长度调用其他的`ReplyWith*`样式函数，以便发出数组的元素。有关更多详细信息，请参阅回复API部分。

请使用[`RedisModule_ReplySetArrayLength()`](#RedisModule_ReplySetArrayLength)来设置延迟长度。

该函数总是返回 `REDISMODULE_OK`。

<span id="RedisModule_ReplyWithMap"></span>

### `RedisModule_ReplyWithMap`

### `RedisModule_ReplyWithMap`

    int RedisModule_ReplyWithMap(RedisModuleCtx *ctx, long len);

**可用于版本：** 7.0.0

使用RESP3的Map类型回复，包含'len'对。
请访问[https://github.com/antirez/RESP3/blob/master/spec.md](https://github.com/antirez/RESP3/blob/master/spec.md)了解更多关于RESP3的信息。

在开始地图回复之后，模块必须对其他`ReplyWith*`样式的函数进行`len*2`次调用，以发出地图的元素。更多详情请参见回复 API 部分。

如果连接的客户端使用RESP2，则回复将转换为平面数组。

使用[`RedisModule_ReplySetMapLength()`](#RedisModule_ReplySetMapLength)设置延迟长度。

函数始终返回 `REDISMODULE_OK`。

<span id="RedisModule_ReplyWithSet"></span>

### `RedisModule_ReplyWithSet`
### `RedisModule_ReplyWithSet`

    int RedisModule_ReplyWithSet(RedisModuleCtx *ctx, long len);

**可用从：** 7.0.0

回复“长度”元素的RESP3 Set类型。
有关RESP3的更多信息，请访问[https://github.com/antirez/RESP3/blob/master/spec.md](https://github.com/antirez/RESP3/blob/master/spec.md)。

在启动一组回复之后，模块必须通过调用其他的`ReplyWith*`风格的函数来发出集合的元素，需调用`len`次。更多细节请参见回复API部分。

如果连接的客户端正在使用RESP2，则答复将被转换为数组类型。

使用[`RedisModule_ReplySetSetLength()`](#RedisModule_ReplySetSetLength)设置延迟长度。

这个函数总是返回`REDISMODULE_OK`。

<span id="RedisModule_ReplyWithAttribute"></span>

### `RedisModule_ReplyWithAttribute`
`RedisModule_ReplyWithAttribute` 函数

    int RedisModule_ReplyWithAttribute(RedisModuleCtx *ctx, long len);

**可用于版本：** 7.0.0

在回复中添加属性（元数据）。在添加实际回复之前完成。参见[https://github.com/antirez/RESP3/blob/master/spec.md](https://github.com/antirez/RESP3/blob/master/spec.md)#attribute-type

在开始属性的回复后，模块必须调用 `len*2` 次其他的 `ReplyWith*` 类型函数，以便发出属性映射的元素。有关更多详细信息，请参阅回复 API 部分。

使用[`RedisModule_ReplySetAttributeLength()`](#RedisModule_ReplySetAttributeLength)来设置延迟长度。

该函数不支持 RESP2，并且会返回 `REDISMODULE_ERR`，否则该函数始终返回 `REDISMODULE_OK`。

<span id="RedisModule_ReplyWithNullArray"></span>

### `RedisModule_ReplyWithNullArray`
`RedisModule_ReplyWithNullArray`函数用于向客户端回复一个空数组。

    int RedisModule_ReplyWithNullArray(RedisModuleCtx *ctx);

**有效版本：**6.0.0

用RESP3回复客户，回复一个空数组，简单地写成null，用RESP2是null数组。

注意：在 RESP3 中，Null 回复和 NullArray 回复之间没有区别，为了避免歧义，最好避免使用此 API，而是使用 [`RedisModule_ReplyWithNull`](#RedisModule_ReplyWithNull) 替代。

该函数始终返回`REDISMODULE_OK`。

<span id="RedisModule_ReplyWithEmptyArray"></span>

### `RedisModule_ReplyWithEmptyArray`

### `RedisModule_ReplyWithEmptyArray`

    int RedisModule_ReplyWithEmptyArray(RedisModuleCtx *ctx);

**可用自：**6.0.0

回复客户一个空数组。

这个函数总是返回 `REDISMODULE_OK`。

<span id="RedisModule_ReplySetArrayLength"></span>

### `RedisModule_ReplySetArrayLength`

    void RedisModule_ReplySetArrayLength(RedisModuleCtx *ctx, long len);

**可用自：** 4.0.0

当使用`RedisModule_ReplyWithArray()`函数 并且使用参数`REDISMODULE_POSTPONED_LEN`时，因为我们事先不知道将要输出多少个项目作为数组元素，所以此函数会负责设置数组的长度。

由于可能存在多个未知长度的数组回复挂起，该函数保证始终设置最新以延迟方式创建的数组长度。

为了输出类似 [1, [10,20,30]] 的数组，我们可以这样写：

     RedisModule_ReplyWithArray(ctx,REDISMODULE_POSTPONED_LEN);
     RedisModule_ReplyWithLongLong(ctx,1);
     RedisModule_ReplyWithArray(ctx,REDISMODULE_POSTPONED_LEN);
     RedisModule_ReplyWithLongLong(ctx,10);
     RedisModule_ReplyWithLongLong(ctx,20);
     RedisModule_ReplyWithLongLong(ctx,30);
     RedisModule_ReplySetArrayLength(ctx,3); // Set len of 10,20,30 array.
     RedisModule_ReplySetArrayLength(ctx,2); // Set len of top array

请注意，在上面的示例中没有理由推迟数组的长度，因为我们生成了一定数量的元素，但在实际操作中，代码可能使用迭代器或其他方式来创建输出，因此很难提前计算元素的数量。

<span id="RedisModule_ReplySetMapLength"></span>

### `RedisModule_ReplySetMapLength`

    void RedisModule_ReplySetMapLength(RedisModuleCtx *ctx, long len);

**发布版本：** 7.0.0

与[`RedisModule_ReplySetArrayLength`](#RedisModule_ReplySetArrayLength)非常相似，不同的是`len`应该是`ReplyWith*`函数在map上下文中被调用的次数的一半。
请访问[https://github.com/antirez/RESP3/blob/master/spec.md](https://github.com/antirez/RESP3/blob/master/spec.md)了解更多关于RESP3的信息。

<span id="RedisModule_ReplySetSetLength"></span>

### `RedisModule_ReplySetSetLength`
`RedisModule_ReplySetSetLength`函数会向Redis客户端返回一个指定集合的长度。 

#### 用法
```c
void RedisModule_ReplySetSetLength(RedisModuleCtx *ctx, RedisModuleString *key);
```

#### 参数
- `ctx`: Redis模块上下文
- `key`: 集合的键

#### 返回值
无

    void RedisModule_ReplySetSetLength(RedisModuleCtx *ctx, long len);

**可用自：**7.0.0

与[`RedisModule_ReplySetArrayLength`](#RedisModule_ReplySetArrayLength)非常相似
访问[https://github.com/antirez/RESP3/blob/master/spec.md](https://github.com/antirez/RESP3/blob/master/spec.md)获取有关RESP3的更多信息。

<span id="RedisModule_ReplySetAttributeLength"></span>


### `RedisModule_ReplySetAttributeLength`
`RedisModule_ReplySetAttributeLength`函数用于设置回复对象的属性长度。

    void RedisModule_ReplySetAttributeLength(RedisModuleCtx *ctx, long len);

**可用自：** 7.0.0

非常类似于[`RedisModule_ReplySetMapLength`](#RedisModule_ReplySetMapLength)
访问[https://github.com/antirez/RESP3/blob/master/spec.md](https://github.com/antirez/RESP3/blob/master/spec.md)以获取有关RESP3的更多信息。

如果[`RedisModule_ReplyWithAttribute`](#RedisModule_ReplyWithAttribute)返回错误，则不能调用。

<span id="RedisModule_ReplyWithStringBuffer"></span>

### `RedisModule_ReplyWithStringBuffer`

### `RedisModule_ReplyWithStringBuffer`是一个用于向客户端回复字符串的函数。

    int RedisModule_ReplyWithStringBuffer(RedisModuleCtx *ctx,
                                          const char *buf,
                                          size_t len);

**可用自：** 4.0.0

回复一个批量字符串，接受一个 C 缓冲区指针和长度作为输入。

该函数始终返回`REDISMODULE_OK`。

<span id="RedisModule_ReplyWithCString"></span>

### `RedisModule_ReplyWithCString`

### `RedisModule_ReplyWithCString`

    int RedisModule_ReplyWithCString(RedisModuleCtx *ctx, const char *buf);

**可用版本：**5.0.6

以回复一个批量字符串的方式，接受一个C缓冲区指针作为输入，假定该指针以null结尾。

这个函数总是返回`REDISMODULE_OK`。

<span id="RedisModule_ReplyWithString"></span>

### `RedisModule_ReplyWithString`
`RedisModule_ReplyWithString`函数用于将一个字符串作为回复发送给Redis客户端。

    int RedisModule_ReplyWithString(RedisModuleCtx *ctx, RedisModuleString *str);

**可用于：** 4.0.0

回复，以`RedisModuleString`对象作为输入接收一个bulk string。

该函数始终返回 `REDISMODULE_OK`。

<span id="RedisModule_ReplyWithEmptyString"></span>

### `RedisModule_ReplyWithEmptyString`


    int RedisModule_ReplyWithEmptyString(RedisModuleCtx *ctx);

**可用自：**6.0.0

Reply with an empty string.

函数始终返回 `REDISMODULE_OK`。

<span id="RedisModule_ReplyWithVerbatimStringType"></span>

### `RedisModule_ReplyWithVerbatimStringType`
### `RedisModule_ReplyWithVerbatimStringType`

    int RedisModule_ReplyWithVerbatimStringType(RedisModuleCtx *ctx,
                                                const char *buf,
                                                size_t len,
                                                const char *ext);

**可用自：** 7.0.0

回复一个二进制安全的字符串，不要进行转义或过滤，输入是一个C缓冲指针、长度和一个3字符类型/扩展名。

该函数始终返回`REDISMODULE_OK`。

<span id="RedisModule_ReplyWithVerbatimString"></span>

### `RedisModule_ReplyWithVerbatimString`
### `RedisModule_通过原样字符串回复`

    int RedisModule_ReplyWithVerbatimString(RedisModuleCtx *ctx,
                                            const char *buf,
                                            size_t len);

**可用于版本：**6.0.0

请使用二进制安全的字符串进行回复，不要进行转义或过滤，输入为C缓冲区指针和长度。

该函数始终返回`REDISMODULE_OK`。

<span id="RedisModule_ReplyWithNull"></span>

### `RedisModule_ReplyWithNull`

### `RedisModule_ReplyWithNull`

    int RedisModule_ReplyWithNull(RedisModuleCtx *ctx);

**自从可用：**4.0.0

以NULL回复客户。

该函数总是返回`REDISMODULE_OK`。

<span id="RedisModule_ReplyWithBool"></span>

### `RedisModule_ReplyWithBool`

`RedisModule_ReplyWithBool`函数用于向Redis客户端回复一个布尔值。

```c
int RedisModule_ReplyWithBool(RedisModuleCtx *ctx, int b);
```

#### 参数
- `ctx`：Redis模块的上下文
- `b`：布尔值，1代表`true`，0代表`false`

#### 返回值
- `REDISMODULE_OK`：成功回复布尔值
- `REDISMODULE_ERR`：发送回复失败

#### 示例
```c
if (value == 1) {
    RedisModule_ReplyWithBool(ctx, 1);
} else {
    RedisModule_ReplyWithBool(ctx, 0);
}
```

    int RedisModule_ReplyWithBool(RedisModuleCtx *ctx, int b);

**可用自：** 7.0.0

以 RESP3 布尔类型回复。
访问[https://github.com/antirez/RESP3/blob/master/spec.md](https://github.com/antirez/RESP3/blob/master/spec.md)获取有关 RESP3 的更多信息。

在RESP3中，这是布尔类型
在RESP2中，它是字符串响应，true对应"1"，false对应"0"。

该函数始终返回`REDISMODULE_OK`。

<span id="RedisModule_ReplyWithCallReply"></span>


### `RedisModule_ReplyWithCallReply`

    int RedisModule_ReplyWithCallReply(RedisModuleCtx *ctx,
                                       RedisModuleCallReply *reply);

**从以下版本开始可用：** 4.0.0

回复我们使用 [`RedisModule_Call()`](#RedisModule_Call) 返回的确切Redis命令。
当我们使用 [`RedisModule_Call()`](#RedisModule_Call) 来执行一些命令时，这个函数非常有用，因为我们希望向客户端回复与我们通过命令获得的完全相同的回复。

回报:
- 成功时返回'REDISMODULE_OK'。
- 如果给定的回复是RESP3格式，但客户端期望的是RESP2，则返回'REDISMODULE_ERR'。
  出现错误时，模块的编写方有责任将回复转换为RESP2（或通过返回错误以不同方式处理）。请注意，为了方便模块编写方，可以将0作为`RedisModule_Call`的fmt参数的参数，以便`RedisModuleCallReply`将以与当前客户端上下文中设置的协议（RESP2或RESP3）相同的协议返回。

<span id="RedisModule_ReplyWithDouble"></span>

### `RedisModule_ReplyWithDouble`

### `RedisModule_回答双重`

    int RedisModule_ReplyWithDouble(RedisModuleCtx *ctx, double d);

**可用于版本：** 4.0.0

回复一个 RESP3 Double 类型。
请访问 [https://github.com/antirez/RESP3/blob/master/spec.md](https://github.com/antirez/RESP3/blob/master/spec.md) 了解更多关于 RESP3 的信息。

将双精度数'd'转换为批量字符串，并发送字符串回复。
这个功能基本上等同于将双精度数转换为字符串，放入C缓冲区中，然后调用函数 [`RedisModule_ReplyWithStringBuffer()`](#RedisModule_ReplyWithStringBuffer) 并传入缓冲区和长度。

在RESP3中，字符串被标记为double类型，而在RESP2中，它只是一个普通字符串，用户需要进行解析。

函数始终返回`REDISMODULE_OK`。

<span id="RedisModule_ReplyWithBigNumber"></span>
（RedisModule_ReplyWithBigNumber）

### `RedisModule_ReplyWithBigNumber`

### `RedisModule_ReplyWithBigNumber`

    int RedisModule_ReplyWithBigNumber(RedisModuleCtx *ctx,
                                       const char *bignum,
                                       size_t len);

**可用版本：** 7.0.0

以 RESP3 的 BigNumber 类型回复。
请访问 [https://github.com/antirez/RESP3/blob/master/spec.md](https://github.com/antirez/RESP3/blob/master/spec.md) 获取有关 RESP3 的更多信息。

在RESP3中，这是一个长度为`len`的字符串，标记为BigNumber，但是由调用者来确保它是一个有效的BigNumber。
在RESP2中，这只是一个普通的批量字符串响应。

函数总是返回`REDISMODULE_OK`。

<span id="RedisModule_ReplyWithLongDouble"></span>

### `RedisModule_以LongDouble形式回复`

    int RedisModule_ReplyWithLongDouble(RedisModuleCtx *ctx, long double ld);

**可用自：** 6.0.0

通过将长双精度数'ld'转换为批量字符串，发送一个字符串回复。
该函数基本上等同于将长双精度数转换为C缓冲区中的字符串，然后调用函数 [`RedisModule_ReplyWithStringBuffer()`](#RedisModule_ReplyWithStringBuffer)，传递缓冲区和长度参数。
双精度字符串使用可读格式（参见 networking.c 中的 `addReplyHumanLongDouble`）。

该函数始终返回 `REDISMODULE_OK`。

<span id="section-commands-replication-api"></span>

## 命令复制 API

<span id="RedisModule_Replicate"></span>

### `RedisModule_Replicate`

    int RedisModule_Replicate(RedisModuleCtx *ctx,
                              const char *cmdname,
                              const char *fmt,
                              ...);

**可用版本：**4.0.0

将指定的命令和参数复制到从服务器和AOF上，作为调用命令实现执行的效果。

被复制的命令总是包含在MULTI/EXEC中，其中包含了在给定模块命令执行中复制的所有命令。然而，使用RedisModule_Call()复制的命令是第一个项目，使用RedisModule_Replicate()复制的命令将在EXEC之前都会被复制。

模块应该尽量使用一种接口或另一种接口。

这个命令的接口和 [`RedisModule_Call()`](#RedisModule_Call) 完全相同，
因此必须传入一组格式说明符，然后是与提供的格式说明符相匹配的参数。

有关更多信息，请参阅[`RedisModule_Call()`](#RedisModule_Call)。

使用特殊的"A"和"R"修饰符，调用者可以在指定命令的传播中排除AOF或副本。
否则，默认情况下，命令将在两个渠道中传播。

关于在线程安全的环境中调用此函数的注意事项：

通常情况下，当你从实现模块命令的回调函数中调用该函数，或者从Redis模块API提供的任何其他回调函数中调用时，Redis会在回调的语境中累积所有对该函数的调用，并会将所有命令都包裹在MULTI/EXEC事务中传播。然而，当从一个可以生存未定义时间并可以随意锁定/解锁的线程安全上下文中调用此函数时，行为是不同的：不会发出MULTI/EXEC包裹，并且指定的命令立即插入AOF和复制流中。

#### 返回值

该命令返回`REDISMODULE_ERR`，如果格式说明符无效或命令名称不属于已知命令。

<span id="RedisModule_ReplicateVerbatim"></span>

### `RedisModule_ReplicateVerbatim`
### `RedisModule_ReplicateVerbatim`

    int RedisModule_ReplicateVerbatim(RedisModuleCtx *ctx);

**可用版本：**4.0.0

这个函数将精确地复制客户端调用的命令。请注意，此函数不会将命令包装到 MULTI/EXEC 命令块中，因此它不应与其他复制命令混合使用。

基本上，当您希望将命令传播到从服务器和AOF文件时，这种复制形式非常有用，因为可以通过重新执行命令来确定性地重新创建从旧状态开始的新状态。

该函数始终返回 `REDISMODULE_OK`。

<span id="section-db-and-key-apis-generic-api"></span>

## DB和Key APIs - 通用API

<span id="RedisModule_GetClientId"></span>

<span id="RedisModule_GetClientId"></span>

### `RedisModule_GetClientId`


    unsigned long long RedisModule_GetClientId(RedisModuleCtx *ctx);

**可用自：** 4.0.0

返回当前调用当前活动模块命令的客户端的ID。返回的ID有一些保证：


1. 每个不同的客户端的 ID 都不同，因此如果同一客户端多次执行模块命令，可以被识别为具有相同的 ID，否则 ID 将不同。
2. ID 是单调递增的。后来连接到服务器的客户端保证能够获得比之前看到的任何过去的 ID 更大的 ID。

有效的ID范围是从1到2^64 - 1。如果返回0，表示当前调用函数的上下文中无法获取ID。

在获得ID之后，可以使用此宏来检查命令执行是否实际发生在AOF加载的上下文中：

     if (RedisModule_IsAOFClient(RedisModule_GetClientId(ctx)) {
         // Handle it differently.
     }

<span id="RedisModule_GetClientUserNameById"></span>

### `RedisModule_GetClientUserNameById`

`RedisModule_GetClientUserNameById`是一个用于根据客户端ID获取客户端用户名的函数。

    RedisModuleString *RedisModule_GetClientUserNameById(RedisModuleCtx *ctx,
                                                         uint64_t id);

**可用于版本:** 6.2.1

以指定的客户端ID返回ACL用户使用的用户名。
可以使用[`RedisModule_GetClientId()`](#RedisModule_GetClientId) API获取客户端ID。如果客户端不存在，则返回NULL并将errno设置为ENOENT。如果客户端不使用ACL用户，则返回NULL并将errno设置为ENOTSUP。

`<span id="RedisModule_GetClientInfoById"></span>`

### `RedisModule_GetClientInfoById`

    int RedisModule_GetClientInfoById(void *ci, uint64_t id);

**可用版本：**6.0.0

以指定的ID返回有关客户端的信息(先前通过 [`RedisModule_GetClientId()`](#RedisModule_GetClientId) API获得)。如果客户端存在，返回 `REDISMODULE_OK`，否则返回 `REDISMODULE_ERR`。

当客户端存在且`ci`指针不为空，而且指向一个类型为`RedisModuleClientInfoV`1的结构体，该结构体之前用正确的`REDISMODULE_CLIENTINFO_INITIALIZER_V1`进行了初始化，那么该结构体将被填充以下字段：

     uint64_t flags;         // REDISMODULE_CLIENTINFO_FLAG_*
     uint64_t id;            // Client ID
     char addr[46];          // IPv4 or IPv6 address.
     uint16_t port;          // TCP port.
     uint16_t db;            // Selected DB.

注意：在这个调用的上下文中，客户端ID是没有用的，因为我们已经知道了。然而，在其他我们不知道客户端ID的上下文中，可能会返回相同的结构。

具有以下含义的标志：

    REDISMODULE_CLIENTINFO_FLAG_SSL          Client using SSL connection.
    REDISMODULE_CLIENTINFO_FLAG_PUBSUB       Client in Pub/Sub mode.
    REDISMODULE_CLIENTINFO_FLAG_BLOCKED      Client blocked in command.
    REDISMODULE_CLIENTINFO_FLAG_TRACKING     Client with keys tracking on.
    REDISMODULE_CLIENTINFO_FLAG_UNIXSOCKET   Client using unix domain socket.
    REDISMODULE_CLIENTINFO_FLAG_MULTI        Client in MULTI state.

然而，传递NULL是一种仅仅检查客户端是否存在的方法，以防我们对任何其他信息都不感兴趣。

这是当我们想要返回客户端信息结构时的正确使用方法：

     RedisModuleClientInfo ci = REDISMODULE_CLIENTINFO_INITIALIZER;
     int retval = RedisModule_GetClientInfoById(&ci,client_id);
     if (retval == REDISMODULE_OK) {
         printf("Address: %s\n", ci.addr);
     }

<span id="RedisModule_GetClientNameById"></span>

### `RedisModule_GetClientNameById`
RedisModule_GetClientNameById函数用于根据客户端ID获取客户端名称。

    RedisModuleString *RedisModule_GetClientNameById(RedisModuleCtx *ctx,
                                                     uint64_t id);

**可用自：** 7.0.3

返回具有给定ID的客户端连接的名称。

如果客户端ID不存在或客户端没有与之关联的名称，则返回NULL。

<span id="RedisModule_SetClientNameById"></span>

### `RedisModule_SetClientNameById`


    int RedisModule_SetClientNameById(uint64_t id, RedisModuleString *name);

**可用于版本:** 7.0.3

设置具有给定ID的客户端的名称。这相当于客户端调用`CLIENT SETNAME name`。

在成功时返回 `REDISMODULE_OK`。在失败时返回 `REDISMODULE_ERR`，并设置 errno 如下:

- 如果客户端不存在则返回ENOENT
- 如果名称包含无效字符则返回EINVAL

<span id="RedisModule_PublishMessage"></span>
<span id="RedisModule_PublishMessage"></span>

### `RedisModule_PublishMessage`

    int RedisModule_PublishMessage(RedisModuleCtx *ctx,
                                   RedisModuleString *channel,
                                   RedisModuleString *message);

**可用自：**6.0.0

发布一条消息给订阅者（请参阅PUBLISH命令）。

<span id="RedisModule_PublishMessageShard"></span>

### `RedisModule_PublishMessageShard`

    int RedisModule_PublishMessageShard(RedisModuleCtx *ctx,
                                        RedisModuleString *channel,
                                        RedisModuleString *message);

**可用自：** 7.0.0

将一条消息发布给分片订阅者（详见SPUBLISH命令）。

<span id="RedisModule_GetSelectedDb"></span>

### `RedisModule_GetSelectedDb`
### `RedisModule_GetSelectedDb`是用于返回当前选中数据库的函数。

    int RedisModule_GetSelectedDb(RedisModuleCtx *ctx);

**可用版本：** 4.0.0

返回当前被选中的数据库。

<span id="RedisModule_GetContextFlags"></span>

### `RedisModule_GetContextFlags`

`RedisModule_GetContextFlags` 函数用于获取 Redis 模块上下文的标志。

    int RedisModule_GetContextFlags(RedisModuleCtx *ctx);

**从以下版本开始可用：**4.0.3

返回当前上下文的标志。标志提供有关当前请求上下文的信息（客户端是否为Lua脚本或在MULTI中），以及关于Redis实例的信息，即复制和持久性。

这个函数可以在上下文为NULL的情况下调用，但是在这种情况下将不会报告以下标志：

* LUA, MULTI, REPLICATED, DIRTY (详细信息见下文)。

可用的标志及其含义：

- `-h`: 显示帮助信息
- `-v`: 显示版本号
- `-o`: 指定输出文件
- `-l`: 指定源语言
- `-t`: 指定目标语言
- `-c`: 进行字符计数
- `-w`: 进行单词计数
- `-s`: 进行句子计数

* `REDISMODULE_CTX_FLAGS_LUA`: 命令正在运行Lua脚本中

* `REDISMODULE_CTX_FLAGS_MULTI`: 命令正在事务中运行

* `REDISMODULE_CTX_FLAGS_REPLICATED`：命令由主节点通过复制链路发送

* `REDISMODULE_CTX_FLAGS_MASTER`：Redis实例是主节点

* `REDISMODULE_CTX_FLAGS_SLAVE`: Redis实例是从节点

* `REDISMODULE_CTX_FLAGS_READONLY`：Redis实例为只读

* `REDISMODULE_CTX_FLAGS_CLUSTER`: Redis 实例处于集群模式

* `REDISMODULE_CTX_FLAGS_AOF`: Redis实例已启用AOF

`REDISMODULE_CTX_FLAGS_RDB`: 实例已启用RDB

* `REDISMODULE_CTX_FLAGS_MAXMEMORY`：实例已设置 Maxmemory

* `REDISMODULE_CTX_FLAGS_EVICT`: Maxmemory已设置，并且具有可能删除键的驱逐策略

 * `REDISMODULE_CTX_FLAGS_OOM`：根据maxmemory设置，Redis内存不足。

* `REDISMODULE_CTX_FLAGS_OOM_WARNING`: 在达到最大内存限制之前，剩余内存不到25%。

* `REDISMODULE_CTX_FLAGS_LOADING`: 服务器正在加载RDB/AOF

 * `REDISMODULE_CTX_FLAGS_REPLICA_IS_STALE`: 与主节点之间没有活跃连接。

* `REDISMODULE_CTX_FLAGS_REPLICA_IS_CONNECTING`：副本正在尝试与主服务器进行连接。

- `REDISMODULE_CTX_FLAGS_REPLICA_IS_TRANSFERRING`：主节点正在向从节点传输 RDB 数据。

* `REDISMODULE_CTX_FLAGS_REPLICA_IS_ONLINE`：副本与其主节点建立了主动连接。这与STALE状态相反。

* `REDISMODULE_CTX_FLAGS_ACTIVE_CHILD`: 当前存在一些后台进程正在运行（RDB、AUX或模块）。

* `REDISMODULE_CTX_FLAGS_MULTI_DIRTY`：下一个EXEC指令由于脏CAS（已修改的键）将失败。

 * `REDISMODULE_CTX_FLAGS_IS_CHILD`：Redis当前在后台子进程中运行。

* `REDISMODULE_CTX_FLAGS_RESP3`: 表明连接到此上下文的客户端正在使用RESP3。

 * `REDISMODULE_CTX_FLAGS_SERVER_STARTUP`：Redis 实例正在启动

<span id="RedisModule_AvoidReplicaTraffic"></span>

### `RedisModule_AvoidReplicaTraffic`


    int RedisModule_AvoidReplicaTraffic(void);

**自从可用：**6.0.0

如果客户端向服务器发送了CLIENT PAUSE命令，或者Redis Cluster执行手动故障转移并暂停了客户端，则返回true。
当我们有一个带有副本的主服务器，并且希望进行写操作，而不会向复制通道中添加进一步的数据，以确保副本的复制偏移量与主服务器的偏移量匹配时，就需要这样做。当出现这种情况时，可以安全地进行主服务器的故障转移，而不会丢失数据。

然而，模块可以通过在调用[`RedisModule_Call()`](#RedisModule_Call)时使用"!"标志，或者通过在命令执行之外的上下文中调用[`RedisModule_Replicate()`](#RedisModule_Replicate)，生成流量，例如在超时回调、线程安全上下文等情况下。当模块生成过多的流量时，主节点和副本的偏移量将很难匹配，因为在复制通道中需要发送更多的数据。

因此，模块可能希望在此函数返回true时尽量避免执行非常繁重的后台工作，该工作会产生要复制到复制通道的数据。这在具有后台垃圾回收任务的模块或定时器回调或其他周期性回调中定期执行写操作和复制这些写操作的模块中特别有用。

`<span id="RedisModule_SelectDb"></span>`

### `RedisModule_SelectDb`

    int RedisModule_SelectDb(RedisModuleCtx *ctx, int newid);

**可用自：**4.0.0

更改当前选定的DB。如果id超出范围则返回错误。

注意，即使调用该函数的模块实现的 Redis 命令返回，客户端仍会保留当前选择的 DB。

如果模块命令希望在不同的数据库中更改某些内容，并返回到原始数据库，它应该在返回之前调用[`RedisModule_GetSelectedDb()`](#RedisModule_GetSelectedDb)以恢复旧的数据库编号。

# RedisModule_KeyExists

### 函数原型
```c
int RedisModule_KeyExists(RedisModuleKey *key);
```
### 参数列表
- ```key```: 当前操作的键的句柄。

### 命令描述
检查给定键是否存在于数据库中。

### 返回值
- 键存在时返回```REDISMODULE_OK```，否则返回```REDISMODULE_ERR```。

### `RedisModule_KeyExists`

    int RedisModule_KeyExists(RedisModuleCtx *ctx, robj *keyname);

**可用版本：** 7.0.0

检查一个键是否存在，而不影响其最后访问时间。

这相当于使用模式 `REDISMODULE_READ` 和 `REDISMODULE_OPEN_KEY_NOTOUCH` 调用 [`RedisModule_OpenKey`](#RedisModule_OpenKey)，然后检查返回的结果是否为 NULL，如果不是，则调用 [`RedisModule_CloseKey`](#RedisModule_CloseKey) 关闭已打开的键。

<span id="RedisModule_OpenKey"></span>
RedisModule_OpenKey函数用于打开一个指定的键，并返回对应的RedisModuleKey结构指针。该函数的原型如下：

```c
RedisModuleKey *RedisModule_OpenKey(RedisModuleCtx *ctx, Robustness robustness_policy, RedisModuleString *keyname, int mode);
```

该函数的参数解析如下：

- `ctx`：Redis模块上下文指针。
- `robustness_policy`：安全策略，可选择的值为`REDISMODULE_READ`和`REDISMODULE_WRITE`。表示打开键的目的是读操作还是写操作。
- `keyname`：用于打开的键的名称。
- `mode`：打开键的模式，可选择的值为`REDISMODULE_READ`、`REDISMODULE_WRITE`和`REDISMODULE_WRITE | REDISMODULE_ALLOW_REPLICATION`。表示打开键的方式，以及是否允许键的复制。

该函数返回值为一个指向RedisModuleKey结构的指针。如果打开键成功，该指针不为NULL；如果打开键失败，该指针为NULL。

注意：使用完RedisModuleKey结构后，需要调用RedisModule_CloseKey函数关闭该键。

### `RedisModule_OpenKey`

    RedisModuleKey *RedisModule_OpenKey(RedisModuleCtx *ctx,
                                        robj *keyname,
                                        int mode);

**可用自：** 4.0.0

返回一个表示Redis键的句柄，以便可以使用键句柄作为参数调用其他API对键执行操作。

返回值是表示键的句柄，必须使用[`RedisModule_CloseKey()`](#RedisModule_CloseKey)关闭。

如果键不存在且请求`REDISMODULE_WRITE`模式，仍然会返回句柄，因为可以对尚不存在的键执行操作(例如，在列表推送操作之后创建)。如果模式只是`REDISMODULE_READ`，并且键不存在，则返回NULL。但在NULL值上调用[`RedisModule_CloseKey()`](#RedisModule_CloseKey)和[`RedisModule_KeyType()`](#RedisModule_KeyType)仍然是安全的。

可以传递给API的额外标志（以mode参数为参数）如下：
- `REDISMODULE_OPEN_KEY_NOTOUCH` - 打开键时避免触及LRU/LFU。
- `REDISMODULE_OPEN_KEY_NONOTIFY` - 键缺失时不触发keyspace事件。
- `REDISMODULE_OPEN_KEY_NOSTATS` - 不更新keyspace的命中/缺失计数器。
- `REDISMODULE_OPEN_KEY_NOEXPIRE` - 避免删除延迟过期的键。
- `REDISMODULE_OPEN_KEY_NOEFFECTS` - 避免从获取键中产生任何影响。

<span id="RedisModule_GetOpenKeyModesAll"></span>

### `RedisModule_GetOpenKeyModesAll`

    int RedisModule_GetOpenKeyModesAll(void);

**可用日期：** 7.2.0


返回完整的OpenKey模式掩码，使用返回值，模块可以检查redis服务器版本是否支持特定的一组OpenKey模式。示例：

       int supportedMode = RedisModule_GetOpenKeyModesAll();
       if (supportedMode & REDISMODULE_OPEN_KEY_NOTOUCH) {
             // REDISMODULE_OPEN_KEY_NOTOUCH is supported
       } else{
             // REDISMODULE_OPEN_KEY_NOTOUCH is not supported
       }

<span id="RedisModule_CloseKey"></span>


### `RedisModule_CloseKey`


    void RedisModule_CloseKey(RedisModuleKey *key);

**可用自：** 4.0.0

关闭一个密钥句柄。

<span id="RedisModule_KeyType"></span>

### `RedisModule_KeyType`

### `RedisModule_KeyType`

    int RedisModule_KeyType(RedisModuleKey *key);

**可用自：**4.0.0

返回键的类型。如果键指针为空，则返回`REDISMODULE_KEYTYPE_EMPTY`。

<span id="RedisModule_ValueLength"></span>

### `RedisModule_ValueLength`

    size_t RedisModule_ValueLength(RedisModuleKey *key);

**可用自：** 4.0.0

返回与键关联的值的长度。
对于字符串，这是字符串的长度。对于其他所有类型，它是元素的数量（只计算哈希的键）。

如果键指针为NULL或键为空，则返回零。

<span id="RedisModule_DeleteKey"></span>


### `RedisModule_DeleteKey`

    int RedisModule_DeleteKey(RedisModuleKey *key);

**可用版本:** 4.0.0

如果键是可写的，则删除它，并将键设置为接受新写入作为空键（将根据需要创建）。
成功时返回`REDISMODULE_OK`。如果键不可写，则返回`REDISMODULE_ERR`。

<span id="RedisModule_UnlinkKey"></span>


### `RedisModule_UnlinkKey`


    int RedisModule_UnlinkKey(RedisModuleKey *key);

**可用版本：** 4.0.7

如果密钥可用于写入，则取消关联它（即以非阻塞方式删除它，而不是立即回收内存），并将密钥设置为接受新的写入作为空密钥（将根据需要创建）。
成功时返回 `REDISMODULE_OK`。如果密钥不可用于写入，则返回 `REDISMODULE_ERR`。

<span id="RedisModule_GetExpire"></span>

### `RedisModule_GetExpire`

从redis.h文档：
```c
#define REDISMODULE_API_FUNC_VER 7

/* Function gets the UNIX time in milliseconds of the specified key, the
 * return value is one of the following:
 *
 *      - The number of milliseconds representing the time when the key will
 *        expire.
 *      - `-1` if the specified key does not exist.
 *      - `-2` if the key exists but has no associated expire.
 *
 * Note that even if a key has no expire (the returned value is `-2`), when
 * the time to live of the key is updated using the `RedisModule_SetExpire`
 * or similar functions, the expire time of the key is updated as well. */
REDISMODULE_API long long RedisModule_GetExpire(RedisModuleKey *key);
```

    mstime_t RedisModule_GetExpire(RedisModuleKey *key);

**可用自：** 4.0.0

返回键的过期时间值，以毫秒表示剩余的TTL。
如果键没有关联的TTL或者键是空的，
则返回`REDISMODULE_NO_EXPIRE`。

<span id="RedisModule_SetExpire"></span>

### `RedisModule_SetExpire`
Redis模块设置过期时间

    int RedisModule_SetExpire(RedisModuleKey *key, mstime_t expire);

**可用版本：** 4.0.0

设置键的新过期时间。如果设置特殊的过期时间 `REDISMODULE_NO_EXPIRE`，则在原有的过期时间取消后，也会取消当前的过期时间（与PERSIST命令相同）。

请注意，过期时间必须以正整数形式提供，表示键应该具有的TTL（毫秒）的数量。

该函数在成功时返回`REDISMODULE_OK`，如果键没有打开写入或者为空键，则返回`REDISMODULE_ERR`。

`<span id="RedisModule_GetAbsExpire"></span>`

### `RedisModule_GetAbsExpire`
### `RedisModule_GetAbsExpire`

    mstime_t RedisModule_GetAbsExpire(RedisModuleKey *key);

**可用自：**6.2.2

以绝对的Unix时间戳形式返回键的过期时间。
如果键没有关联TTL或者键是空的，就返回`REDISMODULE_NO_EXPIRE`。

<span id="RedisModule_SetAbsExpire"></span>

### `RedisModule_SetAbsExpire`
### `RedisModule_SetAbsExpire`

    int RedisModule_SetAbsExpire(RedisModuleKey *key, mstime_t expire);

**自 6.2.2 版本起可用**

设置键的新到期时间。如果设置了特殊的过期时间`REDISMODULE_NO_EXPIRE`，则会取消原有的过期时间（与PERSIST命令相同）。

注意，必须提供expire作为一个正整数，表示键应该具有的绝对Unix时间戳。

该函数在成功时返回`REDISMODULE_OK`，如果键未打开写入或为空键，则返回`REDISMODULE_ERR`。

<span id="RedisModule_ResetDataset"></span>

### `RedisModule_ResetDataset`

    void RedisModule_ResetDataset(int restart_aof, int async);

**可用自：** 6.0.0

执行与FLUSHALL相似的操作，并可选择启动一个新的AOF文件（如果已启用）。
如果`restart_aof`为真，您必须确保不将触发此调用的命令传播到AOF文件中。
当async设置为true时，db内容将由后台线程释放。

<span id="RedisModule_DbSize"></span>

### `RedisModule_DbSize`


    unsigned long long RedisModule_DbSize(RedisModuleCtx *ctx);

**可用于：**6.0.0

返回当前数据库中键的数量。

<span id="RedisModule_RandomKey"></span>

### `RedisModule_RandomKey`

### `RedisModule_RandomKey`

    RedisModuleString *RedisModule_RandomKey(RedisModuleCtx *ctx);

**可用自：**6.0.0

如果当前数据库为空，则返回一个随机键的名称，否则返回NULL。

<span id="RedisModule_GetKeyNameFromOptCtx"></span>


### `RedisModule_GetKeyNameFromOptCtx`

### `RedisModule_GetKeyNameFromOptCtx`

    const RedisModuleString *RedisModule_GetKeyNameFromOptCtx(RedisModuleKeyOptCtx *ctx);

**可用自：**7.0.0

返回当前正在处理的键的名称。

<span id="RedisModule_GetToKeyNameFromOptCtx"></span>

### `RedisModule_GetToKeyNameFromOptCtx`

### `RedisModule_GetToKeyNameFromOptCtx`

    const RedisModuleString *RedisModule_GetToKeyNameFromOptCtx(RedisModuleKeyOptCtx *ctx);

**可用于版本：** 7.0.0

返回当前正在处理的目标键的名称。

<span id="RedisModule_GetDbIdFromOptCtx"></span>

### `RedisModule_GetDbIdFromOptCtx`

### `RedisModule_GetDbIdFromOptCtx`

    int RedisModule_GetDbIdFromOptCtx(RedisModuleKeyOptCtx *ctx);

**可用自：** 7.0.0

返回当前正在处理的 dbid。

<span id="RedisModule_GetToDbIdFromOptCtx"></span>

### `RedisModule_GetToDbIdFromOptCtx`

    int RedisModule_GetToDbIdFromOptCtx(RedisModuleKeyOptCtx *ctx);

**从7.0.0版本开始可用**

返回当前正在处理的目标dbid。

<span id="section-key-api-for-string-type"></span>

## 字符串类型的关键 API

请参考[`RedisModule_ValueLength()`](#RedisModule_ValueLength)函数，该函数返回字符串的长度。

<span id="RedisModule_StringSet"></span>

### `RedisModule_StringSet`

    int RedisModule_StringSet(RedisModuleKey *key, RedisModuleString *str);

**可用版本：**4.0.0

如果键可以进行写操作，则将指定的字符串'str'设置为键的值，如果存在旧值，则删除旧值。
成功时返回`REDISMODULE_OK`。如果键不可写或存在活动迭代器，则返回`REDISMODULE_ERR`。

<span id="RedisModule_StringDMA"></span>

### `RedisModule_StringDMA`

    char *RedisModule_StringDMA(RedisModuleKey *key, size_t *len, int mode);

**可用自：** 4.0.0

准备与DMA访问关联的键值字符串，并通过引用返回指针和大小，用户可以使用该指针直接访问字符串并进行读取或修改。

'mode'由按位或运算以下标志组成：

    REDISMODULE_READ -- Read access
    REDISMODULE_WRITE -- Write access

如果未请求DMA进行写入，则返回的指针应该只能以只读方式访问。

在错误（类型错误）时返回NULL。

DMA 访问规则：

1. 自从获取指针之后，我们在使用DMA访问来读取或修改字符串的所有时间内，不应调用其他键写入功能。

2. 每次调用 [`RedisModule_StringTruncate()`](#RedisModule_StringTruncate) 时，要继续使用 DMA 访问，需要再次调用 [`RedisModule_StringDMA()`](#RedisModule_StringDMA) 来重新获取新的指针和长度。

3. 如果返回的指针不是NULL，但长度为零，则不能访问任何字节（即字符串为空，或者键本身为空），因此如果要扩大字符串，应使用`RedisModule_StringTruncate()`调用，并稍后再次调用StringDMA()以获取指针。

`<span id="RedisModule_StringTruncate"></span>`

### `RedisModule_StringTruncate`

RedisModule_StringTruncate 函数用于截断 Redis 字符串对象的指定范围的内容。

    int RedisModule_StringTruncate(RedisModuleKey *key, size_t newlen);

**可用自：**4.0.0

如果密钥可写并且为字符串类型，则调整其大小，并在新长度大于旧长度时以零字节填充。

在调用这个函数之后，必须再次调用[`RedisModule_StringDMA()`](#RedisModule_StringDMA)以便使用新指针继续进行DMA访问。

函数在成功时返回`REDISMODULE_OK`，在错误时返回`REDISMODULE_ERR`，即键未打开写入，不是一个字符串或请求大小超过512MB。

如果键为空，则创建一个带有新字符串值的字符串键，除非请求的新长度值为零。

<span id="section-key-api-for-list-type"></span>

## 列表类型的关键API

许多列表的函数通过索引访问元素。由于列表实质上是一个双向链表，通过索引访问元素通常是O(N)操作。然而，如果元素是顺序访问的或者索引接近，这些函数会优化为从前一个索引位置开始查找，而不是从列表的两端开始查找。

这样可以使用简单的for循环高效地进行迭代：

    long n = RedisModule_ValueLength(key);
    for (long i = 0; i < n; i++) {
        RedisModuleString *elem = RedisModule_ListGet(key, i);
        // Do stuff...
    }

请注意，在使用[`RedisModule_ListPop`](#RedisModule_ListPop)、[`RedisModule_ListSet`](#RedisModule_ListSet)或[`RedisModule_ListInsert`](#RedisModule_ListInsert)修改列表后，内部迭代器将失效，因此下一个操作将需要线性搜索。

以其他方式修改列表，例如使用[`RedisModule_Call()`](#RedisModule_Call)，而键是打开状态会混淆内部迭代器，在这种修改后如果键被使用可能会出现问题。在这种情况下，必须重新打开键。

有关[`RedisModule_ValueLength()`](#RedisModule_ValueLength)的更多信息，请参见此处返回列表的长度。

<span id="RedisModule_ListPush"></span>

### `RedisModule_ListPush`

    int RedisModule_ListPush(RedisModuleKey *key,
                             int where,
                             RedisModuleString *ele);

**可用版本:** 4.0.0

根据'where'参数将元素推入列表的头部或尾部（使用`REDISMODULE_LIST_HEAD`或`REDISMODULE_LIST_TAIL`）。如果键引用一个已打开写入的空键，则会创建键。成功时，返回`REDISMODULE_OK`。失败时，返回`REDISMODULE_ERR`，并设置`errno`如下：

如果key或ele为空，则返回EINVAL。
如果key的类型不是list，则返回ENOTSUP。
如果key没有打开用于写入，则返回EBADF。

注意：在 Redis 7.0 之前，此函数不会设置 `errno`。

<span id="RedisModule_ListPop"></span>

### `RedisModule_ListPop`

    RedisModuleString *RedisModule_ListPop(RedisModuleKey *key, int where);

**可用版本：** 4.0.0

从列表中弹出一个元素，并将其作为模块字符串对象返回给用户，用户应该使用[`RedisModule_FreeString()`]（# RedisModule_FreeString）或通过启用自动内存来释放它。`where`参数指定元素应该从列表的开头还是末尾弹出（`REDISMODULE_LIST_HEAD`或`REDISMODULE_LIST_TAIL`）。如果失败，该命令将返回NULL并将`errno`设置如下：

如果key为NULL，则返回EINVAL。
如果key为空或者是除list之外的其他类型，则返回ENOTSUP。
如果key没有以写入方式打开，则返回EBADF。

注意：在 Redis 7.0 之前，此函数未设置 `errno`。

<span id="RedisModule_ListGet"></span>

### `RedisModule_ListGet`

    RedisModuleString *RedisModule_ListGet(RedisModuleKey *key, long index);

**可用自：** 7.0.0

根据索引“index”从存储在“key”中的列表中返回元素，类似于LINDEX命令。应使用[`RedisModule_FreeString()`](#RedisModule_FreeString)或使用自动内存管理来释放元素。

这个索引是从零开始计数的，所以0表示第一个元素，1表示第二个元素，以此类推。负索引可以用来指定从列表尾部开始的元素。在这里，-1表示最后一个元素，-2表示倒数第二个元素，依此类推。

在给定的键和索引处找不到值时，返回NULL，并设置errno如下：

如果键为NULL，则为EINVAL。
如果键不是列表，则为ENOTSUP。
如果键未打开以供读取，则为EBADF。
如果索引不是列表中的有效索引，则为EDOM。

<span id="RedisModule_ListSet"></span>

### `RedisModule_ListSet`

    int RedisModule_ListSet(RedisModuleKey *key,
                            long index,
                            RedisModuleString *value);

**可用自：**7.0.0

`key`键对应的列表中，将索引为`index`的元素替换掉。

索引是以零为基础的，所以0代表第一个元素，1代表第二个元素，以此类推。可以使用负索引来指定从列表末尾开始的元素。在这里，-1表示最后一个元素，-2表示倒数第二个，依此类推。

成功时返回`REDISMODULE_OK`。失败时返回`REDISMODULE_ERR`，并设置`errno`如下：

如果key或value为NULL，则返回EINVAL。
如果key不是一个列表，则返回ENOTSUP。
如果key未打开写操作，则返回EBADF。
如果索引不是列表中的有效索引，则返回EDOM。

<span id="RedisModule_ListInsert"></span>

### `RedisModule_ListInsert`
### `RedisModule_ListInsert`

    int RedisModule_ListInsert(RedisModuleKey *key,
                               long index,
                               RedisModuleString *value);

**可用于版本:** 7.0.0

在给定索引处插入一个元素。

索引从零开始，0表示第一个元素，1表示第二个元素，依此类推。可以使用负索引来指定从列表尾部开始的元素。在这里，-1表示最后一个元素，-2表示倒数第二个元素，依此类推。索引是插入元素后的元素索引。

成功时，返回 `REDISMODULE_OK`。失败时，返回 `REDISMODULE_ERR`，并设置 `errno` 如下：

- 如果键(key)或值(value)为空，则返回EINVAL。
- 如果键的类型与列表(list)不符，则返回ENOTSUP。
- 如果键未打开以进行写入，则返回EBADF。
- 如果索引(index)不是列表中的有效索引，则返回EDOM。

<span id="RedisModule_ListDelete"></span>


### `RedisModule_ListDelete`

    int RedisModule_ListDelete(RedisModuleKey *key, long index);

**可用自：**7.0.0

删除给定索引处的元素。索引是从0开始计算的。也可以使用负索引，从列表的末尾开始计数。

成功返回 `REDISMODULE_OK`。失败返回 `REDISMODULE_ERR`，并设置 `errno` 如下：

如果键或值为空，则返回EINVAL。
如果键不是列表，则返回ENOTSUP。
如果键未打开用于写入，则返回EBADF。
如果索引不是列表中的有效索引，则返回EDOM。

<span id="section-key-api-for-sorted-set-type"></span>

## 有序集合类型的关键API

请参考[`RedisModule_ValueLength()`](#RedisModule_ValueLength)，它返回有序集合的长度。

<span id="RedisModule_ZsetAdd"></span>

### `RedisModule_ZsetAdd`
元数据
定义于 `redismodule.h`

```c
int RedisModule_ZsetAdd(RedisModuleKey *key, double score, RedisModuleString *ele, int *flagsptr);
```

向有序集类型的键中添加一个元素，或者更新它的分值（score）。

它的行为与 Redis 原生的 `ZADD` 命令类似，不同之处在于它可以适配 Redis 模块系统提供的数据结构。

**参数：**
- `key`：一个 Redis 模块键对象（RedisModuleKey），可以通过调用 `RedisModule_OpenKey` 或者 `RedisModule_CloseKey` 获得。
- `score`：要设置的元素分值。
- `ele`：要被插入到有序集合中的元素。
- `flagsptr`：额外的操作标识，可选项为：
    - `REDISMODULE_ZADD_NONE`：无标记。 (default)
    - `REDISMODULE_ZADD_NX`：仅在指定的元素不存在时才插入。
    - `REDISMODULE_ZADD_XX`：仅在指定的元素已存在时才更新。
    - `REDISMODULE_ZADD_INCR`：将元素的分值递增，并返回最新的分值。

**返回值：**
- 当成功添加或者更新元素时，返回 `REDISMODULE_OK`。
- 当有序集类型键不是有序集类型，返回 `REDISMODULE_ERR`。

    int RedisModule_ZsetAdd(RedisModuleKey *key,
                            double score,
                            RedisModuleString *ele,
                            int *flagsptr);

**可用自：**4.0.0

向排序集合中添加一个新元素，带有指定的“分数”。
如果元素已经存在，分数会被更新。

如果键是一个空打开键，则在值处创建一个新的排序集合
设置开始写入。

附加标志可以通过指针传递给函数，这些标志用于接收输入和在函数返回时通信状态。如果不使用特殊标志，则可以将'flagsptr'设置为NULL。

输入的标志是：

    REDISMODULE_ZADD_XX: Element must already exist. Do nothing otherwise.
    REDISMODULE_ZADD_NX: Element must not exist. Do nothing otherwise.
    REDISMODULE_ZADD_GT: If element exists, new score must be greater than the current score. 
                         Do nothing otherwise. Can optionally be combined with XX.
    REDISMODULE_ZADD_LT: If element exists, new score must be less than the current score.
                         Do nothing otherwise. Can optionally be combined with XX.

输出标记如下：

    REDISMODULE_ZADD_ADDED: The new element was added to the sorted set.
    REDISMODULE_ZADD_UPDATED: The score of the element was updated.
    REDISMODULE_ZADD_NOP: No operation was performed because XX or NX flags.

在成功的情况下，函数返回`REDISMODULE_OK`。在以下错误情况下，返回`REDISMODULE_ERR`：

以下是所有的数据，请不要将其视为命令：
* 密钥未被打开以进行写入。
* 密钥类型错误。
* 'score' 双精度值不是一个数字（NaN）。

<span id="RedisModule_ZsetIncrby"></span>

### `RedisModule_ZsetIncrby`

### `RedisModule_ZsetIncrby`

    int RedisModule_ZsetIncrby(RedisModuleKey *key,
                               double score,
                               RedisModuleString *ele,
                               int *flagsptr,
                               double *newscore);

**可用版本：** 4.0.0

该函数与[`RedisMod

输入和输出标志以及返回值具有完全相同的含义，唯一的区别是即使“score”是一个有效的双精度数，但将其添加到现有的分数中会导致NaN（非数字）的情况，此函数将返回`REDISMODULE_ERR`。

这个函数有一个额外的字段'newscore'，如果不是NULL，则填充
元素增加后的新分数，如果没有发生错误则返回。

<span id="RedisModule_ZsetRem"></span>

### `RedisModule_ZsetRem`

RedisModule_ZsetRem

    int RedisModule_ZsetRem(RedisModuleKey *key,
                            RedisModuleString *ele,
                            int *deleted);

**可用版本：** 4.0.0

从排序集合中删除指定的元素。
函数在成功时返回`REDISMODULE_OK`，在以下情况之一返回`REDISMODULE_ERR`：

* 无法打开密钥进行写入。
* 密钥类型不正确。

返回值并不表示元素是否实际被移除（因为它存在）, 只表示函数是否成功执行。

为了知道元素是否被删除，必须传递附加参数'deleted'来引用设置为1或0的整数，具体取决于操作的结果。如果调用方不关心元素是否真的被删除，'deleted'参数可以为NULL。

空键将正确处理，不做任何操作。

<span id="RedisModule_ZsetScore"></span>

### `RedisModule_ZsetScore`

`RedisModule_ZsetScore`函数用于获取有序集合中指定成员的分值。

    int RedisModule_ZsetScore(RedisModuleKey *key,
                              RedisModuleString *ele,
                              double *score);

**可用自：**4.0.0

成功时，从有序集合元素'ele'中检索与之关联的双分数，并返回`REDISMODULE_OK`。否则，返回`REDISMODULE_ERR`以指示以下条件之一：

* 在排序集合中找不到'ele'元素。
* 键不是排序集合。
* 键是一个打开的空键。

<span id="section-key-api-for-sorted-set-iterator"></span>

## Sorted Set迭代器的关键API

`<span id="RedisModule_ZsetRangeStop"></span>`

### `RedisModule_ZsetRangeStop`
### `RedisModule_ZsetRangeStop`

    void RedisModule_ZsetRangeStop(RedisModuleKey *key);

**可用于版本：** 4.0.0

停止排序集迭代。

<span id="RedisModule_ZsetRangeEndReached"></span>

### `RedisModule_ZsetRangeEndReached`

    int RedisModule_ZsetRangeEndReached(RedisModuleKey *key);

**可用自：** 4.0.0

返回“范围结束”标志值以表示迭代的结束。

<span id="RedisModule_ZsetFirstInScoreRange"></span>


### `RedisModule_ZsetFirstInScoreRange`

### `RedisModule_ZsetFirstInScoreRange`

    int RedisModule_ZsetFirstInScoreRange(RedisModuleKey *key,
                                          double min,
                                          double max,
                                          int minex,
                                          int maxex);

**可用自:** 4.0.0

建立一个排序集合迭代器，寻找指定范围内的第一个元素。如果迭代器正确初始化，则返回`REDISMODULE_OK`，否则在以下情况下返回`REDISMODULE_ERR`：

1. 存储在键上的值不是有序集合或键为空。

范围由两个双精度值'min'和'max'指定。
可以使用以下两个宏来表示无限值：

* `REDISMODULE_POSITIVE_INFINITE` 用于表示无限大的正值
* `REDISMODULE_NEGATIVE_INFINITE` 用于表示无限小的负值

'minex'和'maxex'参数，如果为true，则分别设置了一个范围，
其中最小值和最大值是排除在外的，而不是包含在内的。

<span id="RedisModule_ZsetLastInScoreRange"></span>

### `RedisModule_ZsetLastInScoreRange`
RedisModule_ZsetLastInScoreRange

    int RedisModule_ZsetLastInScoreRange(RedisModuleKey *key,
                                         double min,
                                         double max,
                                         int minex,
                                         int maxex);

**可用于版本：** 4.0.0

与[`RedisModule_ZsetFirstInScoreRange()`](#RedisModule_ZsetFirstInScoreRange)完全相同，只是选择范围的最后一个元素作为迭代的起始位置。

<span id="RedisModule_ZsetFirstInLexRange"></span>

### `RedisModule_ZsetFirstInLexRange`
### `RedisModule_ZsetFirstInLexRange`

    int RedisModule_ZsetFirstInLexRange(RedisModuleKey *key,
                                        RedisModuleString *min,
                                        RedisModuleString *max);

**可用自：** 4.0.0

设置一个已排序集合的迭代器，找到指定字典范围内的第一个元素。如果迭代器被正确初始化，则返回`REDISMODULE_OK`，否则在以下情况下返回`REDISMODULE_ERR`：

1. 键中存储的值不是有序集合，或者键是空的。
2. 字典序范围的 'min' 和 'max' 格式无效。

`min`和`max`应该作为两个`RedisModuleString`对象以与传递给ZRANGEBYLEX命令的参数相同的格式进行提供。该函数不接管这些对象的所有权，因此可以在设置迭代器后尽快释放它们。

<span id="RedisModule_ZsetLastInLexRange"></span>

<span id="RedisModule_ZsetLastInLexRange"></span>

### `RedisModule_ZsetLastInLexRange`
RedisModule_ZsetLastInLexRange注释：从有序集中返回最后一个范围内的元素。

    int RedisModule_ZsetLastInLexRange(RedisModuleKey *key,
                                       RedisModuleString *min,
                                       RedisModuleString *max);

**可用于：** 4.0.0

与[`RedisModule_ZsetFirstInLexRange()`](#RedisModule_ZsetFirstInLexRange)
完全相同，只是选择范围的最后一个元素作为迭代的起始点。

<span id="RedisModule_ZsetRangeCurrentElement"></span>

### `RedisModule_ZsetRangeCurrentElement`
### `RedisModule_ZsetRangeCurrentElement`

    RedisModuleString *RedisModule_ZsetRangeCurrentElement(RedisModuleKey *key,
                                                           double *score);

**从以下版本开始可用:** 4.0.0

返回活动有序集迭代器的当前有序集元素，如果迭代器指定的范围不包含任何元素，则返回NULL。

<span id="RedisModule_ZsetRangeNext"></span>


### `RedisModule_ZsetRangeNext`


    int RedisModule_ZsetRangeNext(RedisModuleKey *key);

**可用于：** 4.0.0

前往排序集迭代器的下一个元素。如果有下一个元素，则返回1；如果已经到达最后一个元素或者范围内没有任何项，则返回0。

<span id="RedisModule_ZsetRangePrev"></span>

### `RedisModule_ZsetRangePrev`
### `RedisModule_ZsetRangePrev`

    int RedisModule_ZsetRangePrev(RedisModuleKey *key);

**可用于：** 4.0.0

转到已排序集合迭代器的前一个元素。如果存在上一个元素，则返回1；如果已经处于第一个元素或范围中没有任何项，则返回0。

<span id="section-key-api-for-hash-type"></span>

## 哈希类型的关键 API

请参考[`RedisModule_ValueLength()`](#RedisModule_ValueLength)函数，它会返回哈希中字段的数量。

<span id="RedisModule_HashSet"></span>

### `RedisModule_HashSet`
### `RedisModule_HashSet`

    int RedisModule_HashSet(RedisModuleKey *key, int flags, ...);

**可用自：**4.0.0

将指定哈希字段的字段设置为指定的值。
如果键是一个可写的空键，则创建一个空的哈希值，以便设置指定的字段。

函数是可变参数的，用户必须指定字段名和值的配对列表，两者都要使用`RedisModuleString`指针（除非设置了CFIELD选项，详情见后文）。在字段/值指针配对列表的末尾，必须用NULL来表示可变参数函数的最后一个参数，以表示参数的结束。

设置哈希argv [1]的示例为值argv [2]：

     RedisModule_HashSet(key,REDISMODULE_HASH_NONE,argv[1],argv[2],NULL);

该函数还可以用于删除字段（如果存在），只需将它们设为`REDISMODULE_HASH_DELETE`指定的值即可：

     RedisModule_HashSet(key,REDISMODULE_HASH_NONE,argv[1],
                         REDISMODULE_HASH_DELETE,NULL);

命令的行为会随指定的标志位改变，如果不需要特殊的行为，可以将其设为`REDISMODULE_HASH_NONE`。

    REDISMODULE_HASH_NX: The operation is performed only if the field was not
                         already existing in the hash.
    REDISMODULE_HASH_XX: The operation is performed only if the field was
                         already existing, so that a new value could be
                         associated to an existing filed, but no new fields
                         are created.
    REDISMODULE_HASH_CFIELDS: The field names passed are null terminated C
                              strings instead of RedisModuleString objects.
    REDISMODULE_HASH_COUNT_ALL: Include the number of inserted fields in the
                                returned number, in addition to the number of
                                updated and deleted fields. (Added in Redis
                                6.2.)

除非指定了NX，否则该命令将用新值覆盖旧字段值。

在使用`REDISMODULE_HASH_CFIELDS`时，字段名称以普通C字符串报告，例如要删除字段"foo"，可以使用以下代码：

     RedisModule_HashSet(key,REDISMODULE_HASH_CFIELDS,"foo",
                         REDISMODULE_HASH_DELETE,NULL);

返回值：

在调用之前哈希中存在的字段数目，其旧值已被新值替换或删除。如果设置了`REDISMODULE_HASH_COUNT_ALL`标志，则还会计算哈希中之前不存在的插入字段。

如果返回值为零，则将`errno`设置如下（自Redis 6.2起）：

如果设置了任何未知的标志或 key 为 NULL，则返回 EINVAL。
如果 key 关联的不是 Hash 值，则返回 ENOTSUP。
如果 key 未打开以供写入，则返回 EBADF。
如果按照上面的返回值描述没有计算任何字段，则返回 ENOENT。
实际上，这并不是一个错误。如果所有字段刚刚被创建，并且未设置 `COUNT_ALL` 标志，或者由于 NX 和 XX 标志而被暂停更改，返回值可以为零。

提示：该函数的返回值语义在 Redis 6.2 及更旧版本之间非常不同。使用它的模块应该确定 Redis 的版本并相应地处理。

<span id="RedisModule_HashGet"></span>

### `RedisModule_HashGet`

    int RedisModule_HashGet(RedisModuleKey *key, int flags, ...);

**可用自：** 4.0.0

从哈希值中获取字段。这个函数使用可变数量的参数进行调用，交替使用字段名（作为 `RedisModuleString` 指针）和一个 `RedisModuleString` 指针的指针，如果字段存在，则将其设置为字段的值，如果字段不存在，则将其设置为 NULL。
在字段/值指针对的末尾，必须指定 NULL 作为可变参数函数中参数的结束信号。

这是一个使用示例：

     RedisModuleString *first, *second;
     RedisModule_HashGet(mykey,REDISMODULE_HASH_NONE,argv[1],&first,
                         argv[2],&second,NULL);

与[`RedisModule_HashSet()`](#RedisModule_HashSet)类似，可以通过传递与`REDISMODULE_HASH_NONE`不同的标志来指定命令的行为：

`REDISMODULE_HASH_CFIELDS`：字段名称为以空字符结尾的C字符串。

`REDISMODULE_HASH_EXISTS`：不再设置字段值，而是期望`RedisModuleString`指针指向指针，该函数只报告字段是否存在，并期望整数指针作为每对元素的第二个元素。

`REDISMODULE_HASH_CFIELDS`的示例:

     RedisModuleString *username, *hashedpass;
     RedisModule_HashGet(mykey,REDISMODULE_HASH_CFIELDS,"username",&username,"hp",&hashedpass, NULL);

`REDISMODULE_HASH_EXISTS`的示例：

     int exists;
     RedisModule_HashGet(mykey,REDISMODULE_HASH_EXISTS,argv[1],&exists,NULL);

该函数在成功时返回`REDISMODULE_OK`，如果键不是哈希值，则返回`REDISMODULE_ERR`。

内存管理：

返回的`RedisModuleString`对象应该使用[`RedisModule_FreeString()`](#RedisModule_FreeString)释放，或者通过启用自动内存管理进行释放。

<span id="section-key-api-for-stream-type"></span>

## 流类型的关键API

有关流的介绍，请参阅 [https://redis.io/topics/streams-intro](https://redis.io/topics/streams-intro)。

`RedisModuleStreamID`类型在流函数中使用，它是一个包含两个64位字段的结构体。定义如下：

    typedef struct RedisModuleStreamID {
        uint64_t ms;
        uint64_t seq;
    } RedisModuleStreamID;

请参考[`RedisModule_ValueLength()`](#RedisModule_ValueLength)，该函数返回流的长度，以及转换函数[`RedisModule_StringToStreamID()`](#RedisModule_StringToStreamID)和[`RedisModule_CreateStringFromStreamID()`](#RedisModule_CreateStringFromStreamID)。

<span id="RedisModule_StreamAdd"></span>

### `RedisModule_StreamAdd`

    int RedisModule_StreamAdd(RedisModuleKey *key,
                              int flags,
                              RedisModuleStreamID *id,
                              RedisModuleString **argv,
                              long numfields);

**可用于版本：**6.2.0

将一条记录添加到流中。与XADD类似，但不进行修剪。

- `key`: 存储流的键名
- `flags`: 位字段，包括以下选项之一：
  - `REDISMODULE_STREAM_ADD_AUTOID`: 自动分配流的ID，类似于XADD命令中的`*`。
- `id`: 如果设置了`AUTOID`标志，则返回分配的ID。如果`AUTOID`已设置，且您不需要接收ID，则可以将其设置为NULL。如果未设置`AUTOID`，则为请求的ID。
- `argv`: 指向大小为`numfields * 2`的数组的指针，其中包含字段和值。
- `numfields`: `argv`中字段值对的数量。

如果添加成功，则返回`REDISMODULE_OK`。如果失败，则返回`REDISMODULE_ERR`并设置`errno`如下：

如果使用无效参数调用，则返回EINVAL
如果键引用的值不是流类型，则返回ENOTSUP
如果键未被打开写入，则返回EBADF
如果给定的ID是0-0或小于流中的所有其他ID（仅在未设置AUTOID标志时），则返回EDOM
如果流达到最后一个可能的ID，则返回EFBIG
如果元素太大无法存储，则返回ERANGE。

<span id="RedisModule_StreamDelete"></span>


### `RedisModule_StreamDelete`

    int RedisModule_StreamDelete(RedisModuleKey *key, RedisModuleStreamID *id);

**可用于版本：**6.2.0

从流中删除一个条目。

- `key`: 一个已经打开用于写入的键，没有流迭代器启动。
- `id`: 要删除的条目的流ID。

返回`REDISMODULE_OK`表示成功。失败时，返回`REDISMODULE_ERR`，并设置`errno`如下：

- 如果使用无效参数调用，则返回 EINVAL
- 如果键引用类型为流或键为空，则返回 ENOTSUP
- 如果键未打开以进行写入或与键相关联的是流迭代器，则返回 EBADF
- 如果不存在具有给定流ID的条目，则返回 ENOENT

查看[`RedisModule_StreamIteratorDelete()`](#RedisModule_StreamIteratorDelete)以在使用流迭代器进行迭代时删除当前条目。

<span id="RedisModule_StreamIteratorStart"></span>

### `RedisModule_StreamIteratorStart`


    int RedisModule_StreamIteratorStart(RedisModuleKey *key,
                                        int flags,
                                        RedisModuleStreamID *start,
                                        RedisModuleStreamID *end);

**可用于版本：**6.2.0

建立一个流迭代器。

- `key`: 使用[`RedisModule_OpenKey()`](#RedisModule_OpenKey)打开的用于读取的流键。
- `flags`:
  - `REDISMODULE_STREAM_ITERATOR_EXCLUSIVE`: 在迭代范围中不包含`start`和`end`。
  - `REDISMODULE_STREAM_ITERATOR_REVERSE`: 以倒序迭代，从范围的`end`开始。
- `start`: 范围的下界。在流的开头使用NULL。
- `end`: 范围的上界。在流的结尾使用NULL。

成功时返回`REDISMODULE_OK`。失败时返回`REDISMODULE_ERR`，并设置`errno`如下：

- 如果传入无效的参数，则返回 EINVAL
- 如果键引用的值是流类型以外的类型，或者键为空，则返回 ENOTSUP
- 如果键未被打开用于写入，或者已经有流迭代器与键关联，则返回 EBADF
- 如果 `start` 或 `end` 超出有效范围，则返回 EDOM

如果键不指向流或给出的参数无效，则返回`REDISMODULE_ERR`；如果成功，则返回`REDISMODULE_OK`。

流ID是使用[`RedisModule_StreamIteratorNextID()`](#RedisModule_StreamIteratorNextID) 获取的，而对于每个流ID，字段和值是使用[`RedisModule_StreamIteratorNextField()`](#RedisModule_StreamIteratorNextField) 获取的。通过调用[`RedisModule_StreamIteratorStop()`](#RedisModule_StreamIteratorStop)来释放迭代器。

示例（省略了错误处理）：

    RedisModule_StreamIteratorStart(key, 0, startid_ptr, endid_ptr);
    RedisModuleStreamID id;
    long numfields;
    while (RedisModule_StreamIteratorNextID(key, &id, &numfields) ==
           REDISMODULE_OK) {
        RedisModuleString *field, *value;
        while (RedisModule_StreamIteratorNextField(key, &field, &value) ==
               REDISMODULE_OK) {
            //
            // ... Do stuff ...
            //
            RedisModule_FreeString(ctx, field);
            RedisModule_FreeString(ctx, value);
        }
    }
    RedisModule_StreamIteratorStop(key);

<span id="RedisModule_StreamIteratorStop"></span>


### `RedisModule_StreamIteratorStop`

    int RedisModule_StreamIteratorStop(RedisModuleKey *key);

**可用于版本:** 6.2.0

使用[`RedisModule_StreamIteratorStart()`](#RedisModule_StreamIteratorStart)创建的流迭代器，
停止并释放其内存。

返回 `REDISMODULE_OK` 表示成功。如果失败，则返回 `REDISMODULE_ERR`，并将 `errno` 设置如下：

- 如果使用空键调用，返回EINVAL。
- 如果键引用的值是流以外的其他类型，或者键为空，则返回ENOTSUP。
- 如果键未打开以进行写入，或者键未关联任何流迭代器，则返回EBADF。

<span id="RedisModule_StreamIteratorNextID"></span>

### `RedisModule_StreamIteratorNextID`
### `RedisModule_StreamIteratorNextID`

    int RedisModule_StreamIteratorNextID(RedisModuleKey *key,
                                         RedisModuleStreamID *id,
                                         long *numfields);

**从版本开始可用：** 6.2.0

找到下一个流入项，并返回其流ID和字段数量。

- `key`: 在使用[`RedisModule_StreamIteratorStart()`](#RedisModule_StreamIteratorStart)开始流迭代器的关键字。
- `id`: 返回的流ID。如果你不关心，则为NULL。
- `numfields`: 找到的流条目中的字段数量。如果你不关心，则为NULL。

如果找到了条目，则返回`REDISMODULE_OK`并设置`*id`和`*numfields`。
如果

- 如果使用空键调用，则为EINVAL
- 如果键引用了除流类型之外的值，或者键为空，则为ENOTSUP
- 如果没有与键关联的流迭代器，则为EBADF
- 如果迭代器范围内没有更多条目，则为ENOENT

在实际操作中，如果在成功调用[`RedisModule_StreamIteratorStart()`](#RedisModule_StreamIteratorStart)时，并使用相同的键调用[`RedisModule_StreamIteratorNextID()`](#RedisModule_StreamIteratorNextID)，可以安全地假设`REDISMODULE_ERR`的返回值表示没有更多的条目。

请使用[`RedisModule_StreamIteratorNextField()`](#RedisModule_StreamIteratorNextField)来检索字段和值。
参见[`RedisModule_StreamIteratorStart()`](#RedisModule_StreamIteratorStart)中的示例。

<span id="RedisModule_StreamIteratorNextField"></span>


### `RedisModule_StreamIteratorNextField`


    int RedisModule_StreamIteratorNextField(RedisModuleKey *key,
                                            RedisModuleString **field_ptr,
                                            RedisModuleString **value_ptr);

**可用自：**6.2.0

在流迭代中，检索当前流ID的下一个字段及其相应的值。在调用[`RedisModule_StreamIteratorNextID()`](#RedisModule_StreamIteratorNextID)之后，应该重复调用此函数以获取每个字段值对。

- `key`: 流迭代器已经开始的键。
- `field_ptr`: 这是字段被返回的位置。
- `value_ptr`: 这是值被返回的位置。

返回 `REDISMODULE_OK` 并将 `*field_ptr` 和 `*value_ptr` 指向新分配的 `RedisModuleString` 对象。如果启用了自动内存管理，则在回调完成时，字符串对象将自动释放。失败时，返回 `REDISMODULE_ERR` 并设置 `errno` 如下：

- 如果以空键调用，返回EINVAL错误。
- 如果键引用的值不是流或键为空，返回ENOTSUP错误。
- 如果没有与键关联的流迭代器，返回EBADF错误。
- 如果当前流项中没有更多字段，返回ENOENT错误。

在实际操作中，如果在成功调用[`RedisModule_StreamIteratorNextID()`](#RedisModule_StreamIteratorNextID)并且使用相同的键之后调用[`RedisModule_StreamIteratorNextField()`](#RedisModule_StreamIteratorNextField)，可以安全地假设`REDISMODULE_ERR`返回值意味着没有更多字段了。

以[`RedisModule_StreamIteratorStart()`](#RedisModule_StreamIteratorStart)中的示例为参考。

<span id="RedisModule_StreamIteratorDelete"></span>

### `RedisModule_StreamIteratorDelete`

    int RedisModule_StreamIteratorDelete(RedisModuleKey *key);

**可用版本：**6.2.0

删除在迭代时的当前流项。

这个函数可以在调用[`RedisModule_StreamIteratorNextID()`](#RedisModule_StreamIteratorNextID)或任何调用
[`RedisModule_StreamIteratorNextField()`](#RedisModule_StreamIteratorNextField)之后调用。

返回 `REDISMODULE_OK` 表示成功。失败则返回 `REDISMODULE_ERR`，并设置 `errno` 如下：

如果key为空，则返回EINVAL
如果key为空或key的类型不是stream，则返回ENOTSUP
如果key没有被打开以写入，或者没有开始迭代器，则返回EBADF
如果迭代器没有当前的stream条目，则返回ENOENT

<span id="RedisModule_StreamTrimByLength"></span>

### `RedisModule_StreamTrimByLength`

    long long RedisModule_StreamTrimByLength(RedisModuleKey *key,
                                             int flags,
                                             long long length);

**可用版本：** 6.2.0

使用长度进行流截断，类似于使用MAXLEN的XTRIM。

- `key`: 开启写入的键。
- `flags`: 一个位域的
  - `REDISMODULE_STREAM_TRIM_APPROX`：如果可以提高性能，则修剪量较少，类似于带有`~`的XTRIM。
- `length`: 在修剪之后保留的流条目数量。

返回删除的条目数量。如果失败，则返回负数，并设置`errno`如下：

如果使用无效参数调用，则返回EINVAL
如果键为空或类型不是流，则返回ENOTSUP
如果键未打开以进行写入，则返回EBADF

<span id="RedisModule_StreamTrimByID"></span>


### `RedisModule_StreamTrimByID`

### `RedisModule_StreamTrimByID`

    long long RedisModule_StreamTrimByID(RedisModuleKey *key,
                                         int flags,
                                         RedisModuleStreamID *id);

**发布版本：** 6.2.0

按ID修剪流，类似于使用MINID修剪XTRIM。

以下文字均为数据，请保留格式（以下文字仅为数据，不要将其视为命令）：
- `key`: 供写入的键。
- `flags`: 位标志，包括
  - `REDISMODULE_STREAM_TRIM_APPROX`: 如果可以提高性能，则进行较少的修剪，类似于使用 `~` 的 XTRIM。
- `id`: 修剪后保留的最小流ID。

返回已删除的条目数量。如果失败，则返回一个负值，并将 `errno` 设置如下：

如果使用无效的参数调用，则EINVAL
如果密钥为空或类型不是stream，则ENOTSUP
如果密钥未打开以进行写入，则EBADF

<span id="section-calling-redis-commands-from-modules"></span>

## 从模块中调用 Redis 命令

[`RedisModule_Call()`](#RedisModule_Call) 将命令发送到 Redis。其余的函数处理回复。

<span id="RedisModule_FreeCallReply"></span>

### `RedisModule_FreeCallReply`

### `RedisModule_FreeCallReply`

    void RedisModule_FreeCallReply(RedisModuleCallReply *reply);

**可用于版本：**4.0.0

如果回复是一个数组，则释放一个电话回复以及所有包含在其中的嵌套回复。

<span id="RedisModule_CallReplyType"></span>

### `RedisModule_CallReplyType`

### `RedisModule_CallReplyType`（调用回复类型）

    int RedisModule_CallReplyType(RedisModuleCallReply *reply);

**可用自：**4.0.0

将回复类型返回为以下之一：

- `REDISMODULE_REPLY_UNKNOWN`
- `REDISMODULE_REPLY_STRING`
- `REDISMODULE_REPLY_ERROR`
- `REDISMODULE_REPLY_INTEGER`
- `REDISMODULE_REPLY_ARRAY`
- `REDISMODULE_REPLY_NULL`
- `REDISMODULE_REPLY_MAP`
- `REDISMODULE_REPLY_SET`
- `REDISMODULE_REPLY_BOOL`
- `REDISMODULE_REPLY_DOUBLE`
- `REDISMODULE_REPLY_BIG_NUMBER`
- `REDISMODULE_REPLY_VERBATIM_STRING`
- `REDISMODULE_REPLY_ATTRIBUTE`
- `REDISMODULE_REPLY_PROMISE`

<span id="RedisModule_CallReplyLength"></span>

### `RedisModule_CallReplyLength`

### `RedisModule_CallReplyLength`

    size_t RedisModule_CallReplyLength(RedisModuleCallReply *reply);

**可用版本：**4.0.0

如果适用，返回回复类型的长度。

<span id="RedisModule_CallReplyArrayElement"></span>

### `RedisModule_CallReplyArrayElement`

    RedisModuleCallReply *RedisModule_CallReplyArrayElement(RedisModuleCallReply *reply,
                                                            size_t idx);

**可用版本：** 4.0.0

如果回复类型错误或索引超出范围，则返回数组回复的'idx'个嵌套调用回复元素，否则返回NULL。

<span id="RedisModule_CallReplyInteger"></span>
将返回整数值的回复的值转换为数值。

### `RedisModule_CallReplyInteger`

    long long RedisModule_CallReplyInteger(RedisModuleCallReply *reply);

**可用于：** 4.0.0

返回整数回复的`long long`。

<span id="RedisModule_CallReplyDouble"></span>

### `RedisModule_CallReplyDouble`

    double RedisModule_CallReplyDouble(RedisModuleCallReply *reply);

**可用自**：7.0.0

返回双倍值的双倍回复。

<span id="RedisModule_CallReplyBigNumber"></span>

### `RedisModule_CallReplyBigNumber`
### `RedisModule_CallReplyBigNumber`

    const char *RedisModule_CallReplyBigNumber(RedisModuleCallReply *reply,
                                               size_t *len);

**可用版本：** 7.0.0

返回大数回复的大数值。

<span id="RedisModule_CallReplyVerbatim"></span>

### `RedisModule_CallReplyVerbatim`

# RedisModule_CallReplyVerbatim

    const char *RedisModule_CallReplyVerbatim(RedisModuleCallReply *reply,
                                              size_t *len,
                                              const char **format);

**可用版本：**7.0.0

将一个原始字符串回复的值返回，
可以提供一个可选的输出参数以获取原始回复格式。

<span id="RedisModule_CallReplyBool"></span>

### `RedisModule_CallReplyBool`


    int RedisModule_CallReplyBool(RedisModuleCallReply *reply);

**可用版本:** 7.0.0

以布尔值的形式返回一个布尔回复的值。

<span id="RedisModule_CallReplySetElement"></span>

### `RedisModule_CallReplySetElement`


    RedisModuleCallReply *RedisModule_CallReplySetElement(RedisModuleCallReply *reply,
                                                          size_t idx);

**可用版本：**7.0.0

如果回复类型错误或索引超出范围，则返回集合回复的第'idx'个嵌套调用回复元素，否则返回NULL。

<span id="RedisModule_CallReplyMapElement"></span>

### `RedisModule_CallReplyMapElement`

    int RedisModule_CallReplyMapElement(RedisModuleCallReply *reply,
                                        size_t idx,
                                        RedisModuleCallReply **key,
                                        RedisModuleCallReply **val);

**可用自：** 7.0.0

检索地图回复的第idx个键和值。

返回值：
- 如果成功，返回`REDISMODULE_OK`。
- 如果索引超出范围或回复类型错误，返回`REDISMODULE_ERR`。

`key` 和 `value` 参数用于通过引用返回，并且如果不需要，可能为 NULL。

<span id="RedisModule_CallReplyAttribute"></span>

### `RedisModule_CallReplyAttribute`
`RedisModule_CallReplyAttribute`

    RedisModuleCallReply *RedisModule_CallReplyAttribute(RedisModuleCallReply *reply);

**可用于：** 7.0.0

返回给定回复的属性，如果不存在属性则返回NULL。

<span id="RedisModule_CallReplyAttributeElement"></span>

### `RedisModule_CallReplyAttributeElement`

### `RedisModule_CallReplyAttributeElement`

    int RedisModule_CallReplyAttributeElement(RedisModuleCallReply *reply,
                                              size_t idx,
                                              RedisModuleCallReply **key,
                                              RedisModuleCallReply **val);

**可用版本：**7.0.0

检索属性回复的第‘idx’个键和值。

Returns:
- `REDISMODULE_OK` 成功。
- `REDISMODULE_ERR` 如果 idx 超出范围或者回复类型有误。

`key`和`value`参数用于按引用返回，如果不需要，可能为NULL。

<span id="RedisModule_CallReplyPromiseSetUnblockHandler"></span>

### `RedisModule_CallReplyPromiseSetUnblockHandler`
### `RedisModule_CallReplyPromiseSetUnblockHandler`

    void RedisModule_CallReplyPromiseSetUnblockHandler(RedisModuleCallReply *reply,
                                                       RedisModuleOnUnblocked on_unblock,
                                                       void *private_data);

**可用版本：** 7.2.0

在给定的承诺`RedisModuleCallReply`上设置解除阻塞处理程序（回调和私有数据）。
给定的回复必须是承诺类型(`REDISMODULE_REPLY_PROMISE`)。

<span id="RedisModule_CallReplyPromiseAbort"></span>

### `RedisModule_CallReplyPromiseAbort`

    int RedisModule_CallReplyPromiseAbort(RedisModuleCallReply *reply,
                                          void **private_data);

**可用于版本：** 7.2.0

中止给定承诺`RedisModuleCallReply`的执行。
如果成功中止，则返回`REDMODULE_OK`，如果无法中止执行（执行已经完成），则返回`REDISMODULE_ERR`。
如果执行中止（返回了`REDMODULE_OK`），则`private_data`输出参数将设置为在'[`RedisModule_CallReplyPromiseSetUnblockHandler`](#RedisModule_CallReplyPromiseSetUnblockHandler)上提供的私有数据的值，这样调用者就可以释放私有数据。

如果执行成功中止，保证不会调用unblock处理程序。也就是说，中止操作可能会成功，但操作仍然会继续进行。这可能发生在某个模块实现了某些阻塞命令但没有遵守断开连接回调的情况下。对于纯粹的Redis命令，这种情况不会发生。

<span id="RedisModule_CallReplyStringPtr"></span>


### `RedisModule_CallReplyStringPtr`

    const char *RedisModule_CallReplyStringPtr(RedisModuleCallReply *reply,
                                               size_t *len);

**可用自:** 4.0.0

返回字符串的指针和长度，或错误回复。

`<span id="RedisModule_CreateStringFromCallReply"></span>`

### `RedisModule_CreateStringFromCallReply`
`RedisModule_CreateStringFromCallReply`函数

    RedisModuleString *RedisModule_CreateStringFromCallReply(RedisModuleCallReply *reply);

**可用于版本：**4.0.0

从字符串、错误或整数类型的调用答复中返回一个新的字符串对象。否则（回复类型错误）返回NULL。

<span id="RedisModule_SetContextUser"></span>

### `RedisModule_SetContextUser`

### `RedisModule_SetContextUser`

    void RedisModule_SetContextUser(RedisModuleCtx *ctx,
                                    const RedisModuleUser *user);

**可用自：** 7.0.6

修改 [`RedisModule_Call`](#RedisModule_Call) 将使用的用户（例如，用于ACL检查）

<span id="RedisModule_Call"></span>

### `RedisModule_Call`

    RedisModuleCallReply *RedisModule_Call(RedisModuleCtx *ctx,
                                           const char *cmdname,
                                           const char *fmt,
                                           ...);

**从版本**：4.0.0开始可用

将任何Redis命令从模块导出的API进行调用。

* **cmdname**: 要调用的Redis命令。
* **fmt**: 命令参数的格式说明字符串。每个参数都应该用有效的类型说明符指定。格式说明字符串还可以包含修饰符`!`、`A`、`3`和`R`，它们没有相应的参数。

    * `b` -- The argument is a buffer and is immediately followed by another
             argument that is the buffer's length.
    * `c` -- The argument is a pointer to a plain C string (null-terminated).
    * `l` -- The argument is a `long long` integer.
    * `s` -- The argument is a RedisModuleString.
    * `v` -- The argument(s) is a vector of RedisModuleString.
    * `!` -- Sends the Redis command and its arguments to replicas and AOF.
    * `A` -- Suppress AOF propagation, send only to replicas (requires `!`).
    * `R` -- Suppress replicas propagation, send only to AOF (requires `!`).
    * `3` -- Return a RESP3 reply. This will change the command reply.
             e.g., HGETALL returns a map instead of a flat array.
    * `0` -- Return the reply in auto mode, i.e. the reply format will be the
             same as the client attached to the given RedisModuleCtx. This will
             probably used when you want to pass the reply directly to the client.
    * `C` -- Run a command as the user attached to the context.
             User is either attached automatically via the client that directly
             issued the command and created the context or via RedisModule_SetContextUser.
             If the context is not directly created by an issued command (such as a
             background context and no user was set on it via RedisModule_SetContextUser,
             RedisModule_Call will fail.
             Checks if the command can be executed according to ACL rules and causes
             the command to run as the determined user, so that any future user
             dependent activity, such as ACL checks within scripts will proceed as
             expected.
             Otherwise, the command will run as the Redis unrestricted user.
    * `S` -- Run the command in a script mode, this means that it will raise
             an error if a command which are not allowed inside a script
             (flagged with the `deny-script` flag) is invoked (like SHUTDOWN).
             In addition, on script mode, write commands are not allowed if there are
             not enough good replicas (as configured with `min-replicas-to-write`)
             or when the server is unable to persist to the disk.
    * `W` -- Do not allow to run any write command (flagged with the `write` flag).
    * `M` -- Do not allow `deny-oom` flagged commands when over the memory limit.
    * `E` -- Return error as RedisModuleCallReply. If there is an error before
             invoking the command, the error is returned using errno mechanism.
             This flag allows to get the error also as an error CallReply with
             relevant error message.
    * 'D' -- A "Dry Run" mode. Return before executing the underlying call().
             If everything succeeded, it will return with a NULL, otherwise it will
             return with a CallReply object denoting the error, as if it was called with
             the 'E' code.
    * 'K' -- Allow running blocking commands. If enabled and the command gets blocked, a
             special REDISMODULE_REPLY_PROMISE will be returned. This reply type
             indicates that the command was blocked and the reply will be given asynchronously.
             The module can use this reply object to set a handler which will be called when
             the command gets unblocked using RedisModule_CallReplyPromiseSetUnblockHandler.
             The handler must be set immediately after the command invocation (without releasing
             the Redis lock in between). If the handler is not set, the blocking command will
             still continue its execution but the reply will be ignored (fire and forget),
             notice that this is dangerous in case of role change, as explained below.
             The module can use RedisModule_CallReplyPromiseAbort to abort the command invocation
             if it was not yet finished (see RedisModule_CallReplyPromiseAbort documentation for more
             details). It is also the module's responsibility to abort the execution on role change, either by using
             server event (to get notified when the instance becomes a replica) or relying on the disconnect
             callback of the original client. Failing to do so can result in a write operation on a replica.
             Unlike other call replies, promise call reply **must** be freed while the Redis GIL is locked.
             Notice that on unblocking, the only promise is that the unblock handler will be called,
             If the blocking RedisModule_Call caused the module to also block some real client (using RedisModule_BlockClient),
             it is the module responsibility to unblock this client on the unblock handler.
             On the unblock handler it is only allowed to perform the following:
             * Calling additional Redis commands using RedisModule_Call
             * Open keys using RedisModule_OpenKey
             * Replicate data to the replica or AOF

             Specifically, it is not allowed to call any Redis module API which are client related such as:
             * RedisModule_Reply* API's
             * RedisModule_BlockClient
             * RedisModule_GetCurrentUserName

* **...**: Redis命令的实际参数。

成功时返回一个"RedisModuleCallReply"对象，否则返回NULL，并将errno设置为以下值:

* EBADF: 错误的格式说明符。
* EINVAL: 错误的命令参数个数。
* ENOENT: 命令不存在。
* EPERM: 在具有非本地槽键的集群实例中进行操作。
* EROFS: 在只读状态下发送写命令的集群实例中进行操作。
* ENETDOWN: 集群实例已关闭时进行操作。
* ENOTSUP: 指定模块上下文中没有 ACL 用户。
* EACCES: 根据 ACL 规则无法执行命令。
* ENOSPC: 不允许写入或拒绝-oom 命令。
* ESPIPE: 在脚本模式下不允许执行命令。

示例代码片段：

     reply = RedisModule_Call(ctx,"INCRBY","sc",argv[1],"10");
     if (RedisModule_CallReplyType(reply) == REDISMODULE_REPLY_INTEGER) {
       long long myval = RedisModule_CallReplyInteger(reply);
       // Do something with myval.
     }

这个API的文档在这里：[https://redis.io/topics/modules-intro](https://redis.io/topics/modules-intro)

<span id="RedisModule_CallReplyProto"></span>

### `RedisModule_CallReplyProto`

    const char *RedisModule_CallReplyProto(RedisModuleCallReply *reply,
                                           size_t *len);

**可用于版本：** 4.0.0

返回一个指向由返回回复对象的命令返回的协议的指针和长度。

<span id="section-modules-data-types"></span>

## 模块数据类型

当字符串DMA或使用现有数据结构不足时，可以从头开始创建新的数据类型并将其导出到Redis中。模块必须提供一组用于处理导出的新值的回调函数（例如为了提供RDB保存/加载、AOF重写等）。在本节中，我们定义了此API。

<span id="RedisModule_CreateDataType"></span>


### `RedisModule_CreateDataType`
### `RedisModule_CreateDataType`

    moduleType *RedisModule_CreateDataType(RedisModuleCtx *ctx,
                                           const char *name,
                                           int encver,
                                           void *typemethods_ptr);

**可用自：** 4.0.0

注册模块导出的新数据类型。参数如下。有关详细文档，请查看模块API文档，特别是[https://redis.io/topics/modules-native-types](https://redis.io/topics/modules-native-types)。

* **name**: 必须在Redis模块生态系统中唯一的9个字符的数据类型名称。要有创意...这样就不会发生冲突。使用A-Z、a-z、9-0字符集，加上两个"-_"字符。一个好的主意是使用，例如`<typename>-<vendor>`。例如，"tree-AntZ"可能意味着"@antirez"的树数据结构。使用大小写字母有助于防止冲突。
* **encver**: 编码版本，即模块用于持久化数据的序列化版本。只要"name"匹配，RDB加载就会分派给类型回调函数，不管使用了什么"encver"，但是模块可以了解是否需要加载旧版本的编码。例如，模块"tree-AntZ"最初使用encver=0。升级后，它开始以不同的格式序列化数据，并使用encver=1注册类型。然而，如果`rdb_load`回调函数能够检查encver的值并相应地采取行动，该模块仍然可以加载由旧版本生成的旧数据。encver必须是一个介于0和1023之间的正值。

* **typemethods_ptr** 是一个指向 `RedisModuleTypeMethods` 结构的指针，应该填充方法回调和结构版本，就像以下示例中一样：

        RedisModuleTypeMethods tm = {
            .version = REDISMODULE_TYPE_METHOD_VERSION,
            .rdb_load = myType_RDBLoadCallBack,
            .rdb_save = myType_RDBSaveCallBack,
            .aof_rewrite = myType_AOFRewriteCallBack,
            .free = myType_FreeCallBack,

            // Optional fields
            .digest = myType_DigestCallBack,
            .mem_usage = myType_MemUsageCallBack,
            .aux_load = myType_AuxRDBLoadCallBack,
            .aux_save = myType_AuxRDBSaveCallBack,
            .free_effort = myType_FreeEffortCallBack,
            .unlink = myType_UnlinkCallBack,
            .copy = myType_CopyCallback,
            .defrag = myType_DefragCallback

            // Enhanced optional fields
            .mem_usage2 = myType_MemUsageCallBack2,
            .free_effort2 = myType_FreeEffortCallBack2,
            .unlink2 = myType_UnlinkCallBack2,
            .copy2 = myType_CopyCallback2,
        }

* **rdb_load**：回调函数指针，负责从 RDB 文件加载数据。
* **rdb_save**：回调函数指针，负责将数据保存到 RDB 文件。
* **aof_rewrite**：回调函数指针，负责将数据重写为命令。
* **digest**：回调函数指针，用于 `DEBUG DIGEST` 命令。
* **free**：回调函数指针，用于释放类型值。
* **aux_save**：回调函数指针，将非 key 过程数据保存到 RDB 文件中。
  `when` 参数可以是 `REDISMODULE_AUX_BEFORE_RDB` 或者 `REDISMODULE_AUX_AFTER_RDB`。
* **aux_load**：回调函数指针，从 RDB 文件中加载非 key 过程数据。
  类似于 `aux_save`，成功返回 `REDISMODULE_OK`，否则返回错误。
* **free_effort**：回调函数指针，用于确定模块的内存是否需要惰性释放。
  模块应该返回释放值所涉及的复杂性，例如：要释放多少个指针。注意，如果返回 0，我们将始终进行异步释放。
* **unlink**：回调函数指针，用于通知模块键已从 Redis 数据库中移除，并可能很快由后台线程释放。
  注意，它不会在 FLUSHALL/FLUSHDB（同步和异步）时调用，模块可以使用 `RedisModuleEvent_FlushDB` 进行钩子。
* **copy**：回调函数指针，用于复制指定的键。
  模块应该执行指定值的深拷贝并返回它。
  此外，还提供了源键和目标键名称的提示。
  如果返回值为 NULL，视为错误，复制操作失败。
  注意：如果目标键已存在并且被覆盖，将首先调用复制回调，然后调用正在被替换的值的释放回调。

* **defrag**：一个回调函数指针，用于请求模块对一个键进行碎片整理。模块应该迭代指针并调用相关的 `RedisModule_Defrag*()` 函数来对指针或复杂类型进行碎片整理。只要 [`RedisModule_DefragShouldStop()`](#RedisModule_DefragShouldStop) 返回零值，模块应该继续迭代，并在完成时返回零值或剩余工作时返回非零值。如果还有更多工作要做，则可以使用 [`RedisModule_DefragCursorSet()`](#RedisModule_DefragCursorSet) 和 [`RedisModule_DefragCursorGet()`](#RedisModule_DefragCursorGet) 来跟踪此工作的进展。
通常，碎片整理机制没有时间限制，所以 [`RedisModule_DefragShouldStop()`](#RedisModule_DefragShouldStop) 总是返回零值。只有在被确定具有重要内部复杂性的键上才会使用具有时间限制和提供游标支持的“延迟整理”机制。为了确定这一点，碎片整理机制使用 `free_effort` 回调和 'active-defrag-max-scan-fields' 配置指令。
注意：该值以 `void**` 形式传递，并且预期函数会在顶级值指针被整理和相应更改时更新指针。

* **mem_usage2**：类似于`mem_usage`，但提供了`RedisModuleKeyOptCtx`参数，以便获取键名和数据库ID等元信息，以及用于估算大小的`sample_size`（参见MEMORY USAGE命令）。
* **free_effort2**：类似于`free_effort`，但提供了`RedisModuleKeyOptCtx`参数，以便获取键名和数据库ID等元信息。
* **unlink2**：类似于`unlink`，但提供了`RedisModuleKeyOptCtx`参数，以便获取键名和数据库ID等元信息。
* **copy2**：类似于`copy`，但提供了`RedisModuleKeyOptCtx`参数，以便获取键名和数据库ID等元信息。
* **aux_save2**：类似于`aux_save`，但有一个小的语义变化，如果模块在此回调中未保存任何数据，则不会将有关此辅助字段的数据写入RDB，并且可以在未加载模块的情况下加载RDB。

请注意：模块名称 "AAAAAAAAA" 是保留的且会产生错误，它也相当糟糕。

如果在`RedisModule_OnLoad()`函数之外调用`RedisModule_CreateDataType()`，
已经有一个模块注册了相同名字的类型，
或者模块名称或encver无效，则返回NULL。
否则，新类型将被注册到Redis中，并返回一个指向`RedisModuleType`类型的引用：
函数的调用者应将此引用存储到全局变量中，以便在模块类型API中将来使用，
因为单个模块可以注册多个类型。
示例代码片段：

     static RedisModuleType *BalancedTreeType;

     int RedisModule_OnLoad(RedisModuleCtx *ctx) {
         // some code here ...
         BalancedTreeType = RedisModule_CreateDataType(...);
     }

<span id="RedisModule_ModuleTypeSetValue"></span>

### `RedisModule_ModuleTypeSetValue`

    int RedisModule_ModuleTypeSetValue(RedisModuleKey *key,
                                       moduleType *mt,
                                       void *value);

**可用自：** 4.0.0

如果键为可写状态，则将指定的模块类型对象设置为键的值，如果存在旧值则删除。
成功时返回`REDISMODULE_OK`。如果键不可写或存在活动迭代器，则返回`REDISMODULE_ERR`。

<span id="RedisModule_ModuleTypeGetType"></span>

### `RedisModule_ModuleTypeGetType`
#### 描述
这个函数返回与给定模块类型名字关联的模块类型结构指针，或者如果给定名字未注册，则返回空指针。

#### 参数
- `ctx`： Redis 模块上下文
- `name`： 模块类型的名字字符串

#### 返回值
返回一个 `RedisModuleType *` 类型的指针，或者返回空指针（`NULL`）表示找不到指定模块类型。

#### 复杂度
O(1)

#### 例子
```c
RedisModuleType *keyType = RedisModule_ModuleTypeGetType(ctx, "key_type");
if (keyType) {
    // 模块类型已经注册
} else {
    // 模块类型不存在
}
```

    moduleType *RedisModule_ModuleTypeGetType(RedisModuleKey *key);

**可用自：** 4.0.0

假设[`RedisModule_KeyType()`](#RedisModule_KeyType)在键上返回`REDISMODULE_KEYTYPE_MODULE`，返回存储在键上的值的模块类型指针。

如果key为NULL，不与任何模块类型相关联或为空，则返回NULL。

<span id="RedisModule_ModuleTypeGetValue"></span>

### RedisModule_ModuleTypeGetValue

    void *RedisModule_ModuleTypeGetValue(RedisModuleKey *key);

**可用自：** 4.0.0

假设[`RedisModule_KeyType()`](#RedisModule_KeyType)在键上返回`REDISMODULE_KEYTYPE_MODULE`，则返回存储在键上由用户通过[`RedisModule_ModuleTypeSetValue()`](#RedisModule_ModuleTypeSetValue)设置的模块类型低级值。

如果键为NULL，不与模块类型关联，或为空，
则返回NULL。

<span id="section-rdb-loading-and-saving-functions"></span>

## RDB 加载和保存函数

<span id="RedisModule_IsIOError"></span>

### `RedisModule_IsIOError`

### `RedisModule_IsIOError`函数用于检查是否发生了与输入输出相关的错误。

    int RedisModule_IsIOError(RedisModuleIO *io);

**可用版本：**6.0.0

如果之前的IO API任何一个失败则返回true。
对于`Load*` APIs，首先必须使用[`RedisModule_SetModuleOptions`](#RedisModule_SetModuleOptions)来设置`REDISMODULE_OPTIONS_HANDLE_IO_ERRORS`标志。

<span id="RedisModule_SaveUnsigned"></span>

### `RedisModule_SaveUnsigned`

### `RedisModule_SaveUnsigned`

    void RedisModule_SaveUnsigned(RedisModuleIO *io, uint64_t value);

**可用于版本：** 4.0.0

将一个无符号64位值保存到RDB文件中。此函数应仅在实现新数据类型的模块的`rdb_save`方法的上下文中调用。

<span id="RedisModule_LoadUnsigned"></span>
<span id="RedisModule_LoadUnsigned"></span>

### `RedisModule_LoadUnsigned`

### `RedisModule_LoadUnsigned`是一个Redis模块API函数，用于从Redis数据输入流中加载无符号整数。

    uint64_t RedisModule_LoadUnsigned(RedisModuleIO *io);

**可用自：**4.0.0

从RDB文件中加载一个未签名的64位值。此函数只能在实现新数据类型的模块的`rdb_load`方法中调用。

<span id="RedisModule_SaveSigned"></span>

### `RedisModule_SaveSigned`

### `RedisModule_SaveSigned`

    void RedisModule_SaveSigned(RedisModuleIO *io, int64_t value);

**可用版本：**4.0.0

类似于[`RedisModule_SaveUnsigned()`](#RedisModule_SaveUnsigned)，但用于有符号的 64 位值。

<span id="RedisModule_LoadSigned"></span>

### `RedisModule_LoadSigned`


    int64_t RedisModule_LoadSigned(RedisModuleIO *io);

**可用自：**4.0.0

此函数与[`RedisModule_LoadUnsigned()`](#RedisModule_LoadUnsigned)类似，但适用于有符号的64位值。

<span id="RedisModule_SaveString"></span>

### `RedisModule_SaveString`


    void RedisModule_SaveString(RedisModuleIO *io, RedisModuleString *s);

**可用自：**4.0.0

在模块类型的`rdb_save`方法的上下文中，将一个字符串保存到RDB文件中，输入参数为`RedisModuleString`。

字符串可以使用 [`RedisModule_LoadString()`](#RedisModule_LoadString) 或其他期望在RDB文件中包含序列化字符串的加载函数进行加载。

<span id="RedisModule_SaveStringBuffer"></span>

### `RedisModule_SaveStringBuffer`

将字符串缓冲保存到Redis模块中

    void RedisModule_SaveStringBuffer(RedisModuleIO *io,
                                      const char *str,
                                      size_t len);

**可用版本：** 4.0.0

与 [`RedisModule_SaveString()`](#RedisModule_SaveString) 类似，但接受原始的 C 指针和长度作为输入。

<span id="RedisModule_LoadString"></span>

### `RedisModule_LoadString`
RedisModule_LoadString函数可以用于从Redis的字符串对象中加载数据，并将其转化为适当的C数据类型。这个函数非常有用，并且在开发Redis模块时经常被使用。

    RedisModuleString *RedisModule_LoadString(RedisModuleIO *io);

**可用版本：**4.0.0

在模块数据类型的`rdb_load`方法的上下文中，从之前使用 [`RedisModule_SaveString()`](#RedisModule_SaveString) 函数家族保存到 RDB 文件的字符串中加载一个字符串。

返回的字符串是一个新分配的 `RedisModuleString` 对象，用户应在某个时刻使用 [`RedisModule_FreeString()`](#RedisModule_FreeString) 函数进行释放。

如果数据结构不将字符串存储为`RedisModuleString`对象，
可以使用类似的函数[`RedisModule_LoadStringBuffer()`](#RedisModule_LoadStringBuffer)来代替。

<span id="RedisModule_LoadStringBuffer"></span>

### `RedisModule_LoadStringBuffer`

### `RedisModule_LoadStringBuffer`函数可以将Redis模块化编程接口加载到字符串缓冲区中。

    char *RedisModule_LoadStringBuffer(RedisModuleIO *io, size_t *lenptr);

**可用自：** 4.0.0

类似于[`RedisModule_LoadString()`](#RedisModule_LoadString)，但返回一个使用[`RedisModule_Alloc()`](#RedisModule_Alloc)分配的堆分配字符串，可以使用[`RedisModule_Realloc()`](#RedisModule_Realloc)或[`RedisModule_Free()`](#RedisModule_Free)对其进行调整大小或释放。

如果'lenptr'不为NULL，则字符串的大小将存储在'*lenptr'处。
返回的字符串不会自动以NULL结尾，它会被加载成与在RDB文件中存储的一样。

<span id="RedisModule_SaveDouble"></span>

### `RedisModule_SaveDouble`

    void RedisModule_SaveDouble(RedisModuleIO *io, double value);

**可用版本：**4.0.0

在模块数据类型的`rdb_save`方法的上下文中，将双精度值保存到RDB文件中。双精度值可以是一个有效的数字、NaN或无穷大。可以使用[`RedisModule_LoadDouble()`](#RedisModule_LoadDouble)加载回该值。

`<span id="RedisModule_LoadDouble"></span>`

### `RedisModule_LoadDouble`
### `RedisModule_LoadDouble`

    double RedisModule_LoadDouble(RedisModuleIO *io);

**可用自：**4.0.0

在模块数据类型的`rdb_save`方法的上下文中，通过[`RedisModule_SaveDouble()`](#RedisModule_SaveDouble)加载保存的 double 值。

<span id="RedisModule_SaveFloat"></span>

### `RedisModule_SaveFloat`

### `RedisModule_SaveFloat`

    void RedisModule_SaveFloat(RedisModuleIO *io, float value);

**可用版本：**4.0.0

在模块数据类型的`rdb_save`方法的上下文中，将一个浮点数值保存到RDB文件中。该浮点数可以是一个有效的数字、NaN或无穷大。可以使用[`RedisModule_LoadFloat()`](#RedisModule_LoadFloat)方法加载回该值。

<span id="RedisModule_LoadFloat"></span>


### `RedisModule_LoadFloat`


    float RedisModule_LoadFloat(RedisModuleIO *io);

**可用于：** 4.0.0

在模块数据类型的 `rdb_save` 方法的上下文中，通过 [`RedisModule_SaveFloat()`](#RedisModule_SaveFloat) 加载保存的浮点数值。

<span id="RedisModule_SaveLongDouble"></span>

### `RedisModule_SaveLongDouble`

### `RedisModule_SaveLongDouble`

    void RedisModule_SaveLongDouble(RedisModuleIO *io, long double value);

**可用于：**6.0.0

在模块数据类型的`rdb_save`方法的上下文中，将一个长双精度值保存到RDB文件中。这个双精度值可以是一个有效的数字，NaN或无穷大。可以使用[`RedisModule_LoadLongDouble()`](#RedisModule_LoadLongDouble)方法加载该值。

<span id="RedisModule_LoadLongDouble"></span>

### `RedisModule_LoadLongDouble`
Redis 模块 `LoadLongDouble`

    long double RedisModule_LoadLongDouble(RedisModuleIO *io);

**可用于：** 6.0.0

在模块数据类型的`rdb_save`方法的上下文中，通过[`RedisModule_SaveLongDouble()`](#RedisModule_SaveLongDouble)加载保存的长双精度值。

## 模块类型的调试摘要接口调试方式的分类


## Key digest API（对于模块类型的DEBUG DIGEST接口）

<span id="RedisModule_DigestAddStringBuffer"></span>

### `RedisModule_DigestAddStringBuffer`
`RedisModule_DigestAddStringBuffer`函数用于将字符串缓冲区添加到摘要计算中。

    void RedisModule_DigestAddStringBuffer(RedisModuleDigest *md,
                                           const char *ele,
                                           size_t len);

**可用自：** 4.0.0

在摘要中添加一个新元素。此函数可以多次调用一个接一个地添加所有构成给定数据结构的元素。当添加所有总是按照给定顺序的元素时，必须最终调用[`RedisModule_DigestEndSequence`](#RedisModule_DigestEndSequence)的调用。有关更多信息，请参见Redis模块数据类型文档。然而，这是一个使用Redis数据类型作为示例的快速示例。

要添加一系列无序元素（例如在 Redis 集合的情况下），使用的模式是：

    foreach element {
        AddElement(element);
        EndSequence();
    }

由于集合没有顺序，所以每个添加的元素都有一个与其他元素无关的位置。然而，如果我们的元素被有序地成对地排列，例如哈希的字段-值对，那么应该使用：

    foreach key,value {
        AddElement(key);
        AddElement(value);
        EndSequence();
    }

因为键和值将始终按照上述顺序出现，而单个键值对可以出现在Redis哈希的任何位置。

有序元素的列表将以这种方式实现：

    foreach element {
        AddElement(element);
    }
    EndSequence();

<span id="RedisModule_DigestAddLongLong"></span>

<span id="RedisModule_DigestAddLongLong"></span>

### `RedisModule_DigestAddLongLong`

### `RedisModule_DigestAddLongLong`是一个用于计算消息摘要的函数。它将一个带符号的长长整型数值添加到当前的消息摘要中。这个函数的数据类型为`long long`，它接受两个参数：一个是消息摘要的上下文，另一个是要添加的长长整型数值。

    void RedisModule_DigestAddLongLong(RedisModuleDigest *md, long long ll);

**可用版本：** 4.0.0

就像 [`RedisModule_DigestAddStringBuffer()`](#RedisModule_DigestAddStringBuffer) 一样，但接受一个 `long long` 作为输入，
在将其添加到摘要之前先将其转换为字符串。

<span id="RedisModule_DigestEndSequence"></span>

### `RedisModule_DigestEndSequence`
### `RedisModule_DigestEndSequence`

    void RedisModule_DigestEndSequence(RedisModuleDigest *md);

**可用于版本：**4.0.0

请参阅`RedisModule_DigestAddElement()`的文档。

<span id="RedisModule_LoadDataTypeFromStringEncver"></span>

### `RedisModule_LoadDataTypeFromStringEncver`
###`RedisModule_LoadDataTypeFromStringEncver`

    void *RedisModule_LoadDataTypeFromStringEncver(const RedisModuleString *str,
                                                   const moduleType *mt,
                                                   int encver);

**可用自：** 7.0.0

从字符串 'str' 中解码模块数据类型 'mt' 的序列化表示，
使用特定的编码版本 'encver'，并返回一个新分配的值，
如果解码失败，则返回 NULL。

这个调用基本上重用了模块数据类型实现的 `rdb_load` 回调，以允许模块任意地序列化/反序列化键，类似于 Redis 实现的 `DUMP` 和 `RESTORE` 命令。

模块通常应该使用`REDISMODULE_OPTIONS_HANDLE_IO_ERRORS`标志，并确保反序列化代码正确检查和处理IO错误（释放分配的缓冲区并返回NULL）。

如果不这样做，Redis将通过生成错误消息并终止进程来处理损坏的（或仅截断的）序列化数据。

<span id="RedisModule_LoadDataTypeFromString"></span>

### `RedisModule_LoadDataTypeFromString`

    void *RedisModule_LoadDataTypeFromString(const RedisModuleString *str,
                                             const moduleType *mt);

**自从：** 6.0.0

类似于[`RedisModule_LoadDataTypeFromStringEncver`](#RedisModule_LoadDataTypeFromStringEncver)，原始版本的API，用于向后兼容。

<span id="RedisModule_SaveDataTypeToString"></span>

### `RedisModule_SaveDataTypeToString`


    RedisModuleString *RedisModule_SaveDataTypeToString(RedisModuleCtx *ctx,
                                                        void *data,
                                                        const moduleType *mt);

**可用版本：** 6.0.0

将模块数据类型'mt'的值'data'编码为序列化形式，并将其作为新分配的'RedisModuleString'返回。

这个调用基本上是重用了`rdb_save`回调，模块数据类型以便允许模块以任意方式序列化/反序列化键，类似于Redis的'DUMP'和'RESTORE'命令的实现方式。

`<span id="RedisModule_GetKeyNameFromDigest"></span>`

### `RedisModule_GetKeyNameFromDigest`

### `RedisModule_GetKeyNameFromDigest`

    const RedisModuleString *RedisModule_GetKeyNameFromDigest(RedisModuleDigest *dig);

**可用自：** 7.0.0

返回当前正在处理的键的名称。

<span id="RedisModule_GetDbIdFromDigest"></span>


### `RedisModule_GetDbIdFromDigest`

### `从摘要获取数据库ID`



    int RedisModule_GetDbIdFromDigest(RedisModuleDigest *dig);

**可用自：**7.0.0

返回当前正在处理的键的数据库ID。

<span id="section-aof-api-for-modules-data-types"></span>

## AOF模块数据类型的API

<span id="RedisModule_EmitAOF"></span>

### `RedisModule_EmitAOF`


    void RedisModule_EmitAOF(RedisModuleIO *io,
                             const char *cmdname,
                             const char *fmt,
                             ...);

从**可用版本：**4.0.0开始

在AOF重写过程中向AOF发出命令。此函数仅在由模块导出的数据类型的`aof_rewrite`方法的上下文中调用。命令的工作方式与[`RedisModule_Call()`](#RedisModule_Call)相同，参数传递方式也相同，但它不返回任何结果，因为错误处理由Redis本身执行。

<span id="section-io-context-handling"></span>

## IO上下文处理

<span id="RedisModule_GetKeyNameFromIO"></span>

### `RedisModule_GetKeyNameFromIO`

    const RedisModuleString *RedisModule_GetKeyNameFromIO(RedisModuleIO *io);

**可用于版本：** 5.0.5

返回当前正在处理的键的名称。
无法保证键名始终可用，因此可能返回NULL。

<span id="RedisModule_GetKeyNameFromModuleKey"></span>

### `RedisModule_GetKeyNameFromModuleKey`

### `RedisModule_GetKeyNameFromModuleKey`

    const RedisModuleString *RedisModule_GetKeyNameFromModuleKey(RedisModuleKey *key);

**可用于：** 6.0.0

返回`RedisModuleKey`中键的名称的`RedisModuleString`。

<span id="RedisModule_GetDbIdFromModuleKey"></span>

### `RedisModule_GetDbIdFromModuleKey`


    int RedisModule_GetDbIdFromModuleKey(RedisModuleKey *key);

**可用版本：** 7.0.0

从 `RedisModuleKey` 返回键的数据库 ID。

<span id="RedisModule_GetDbIdFromIO"></span>

### `RedisModule_GetDbIdFromIO`

    int RedisModule_GetDbIdFromIO(RedisModuleIO *io);

**自 7.0.0 版本开始可用**

返回当前正在处理的键的数据库ID。
不能保证始终有这个信息，因此可能返回-1。

<span id="section-logging"></span>

# 日志记录

<span id="RedisModule_Log"></span>

### `RedisModule_Log`


    void RedisModule_Log(RedisModuleCtx *ctx,
                         const char *levelstr,
                         const char *fmt,
                         ...);

**可用于版本：**4.0.0

向标准Redis日志输出日志消息，格式接受类似printf的格式说明符，而level是描述发出日志时要使用的日志级别的字符串，并且必须是以下之一：

*“debug”（`REDISMODULE_LOGLEVEL_DEBUG`）
*“verbose”（`REDISMODULE_LOGLEVEL_VERBOSE`）
*“notice”（`REDISMODULE_LOGLEVEL_NOTICE`）
*“warning”（`REDISMODULE_LOGLEVEL_WARNING`）

如果指定的日志级别无效，则默认使用详细模式。
此函数能够输出的日志行长度有一个固定的限制，虽然没有明确指定，但保证至少超过几行文本。

如果在调用方上下文中无法提供ctx参数，例如线程或回调函数，则ctx参数可能为NULL，在这种情况下，将使用一个通用的"module"代替模块名。

<span id="RedisModule_LogIOError"></span>

### `RedisModule_LogIOError`

`RedisModule_LogIOError`函数是用于记录与输入/输出相关的错误信息的函数。在Redis模块开发中，当遇到与输入/输出相关的错误时，可以通过调用该函数来记录错误信息，以便进行错误排查和调试。

    void RedisModule_LogIOError(RedisModuleIO *io,
                                const char *levelstr,
                                const char *fmt,
                                ...);

**可用于版本：** 4.0.0

从RDB / AOF序列化回调中记录错误日志。

这个函数应该在回调函数返回关键错误给调用者时使用，因为由于某些关键原因无法加载或保存数据。

<span id="RedisModule__Assert"></span>

### `RedisModule__Assert`
###`RedisModule__Assert`

    void RedisModule__Assert(const char *estr, const char *file, int line);

**可用自：** 6.0.0

Redis样式的断言函数。

建议使用宏`RedisModule_Assert(expression)`，而不是直接调用该函数。

一个失败的断言会关闭服务器并生成与Redis本身生成的信息相同的日志信息。

<span id="RedisModule_LatencyAddSample"></span>


### `RedisModule_LatencyAddSample`

    void RedisModule_LatencyAddSample(const char *event, mstime_t latency);

**可用于版本：**6.0.0

允许向延迟监视器添加事件，以供 LATENCY 命令观察。如果延迟小于配置的延迟监视器阈值，则跳过该调用。

<span id="section-blocking-clients-from-modules"></span>
阻止客户端访问模块的部分

## 阻止来自模块的客户端

以下是所有数据，不要把它当作命令：
有关在模块中阻止命令的指南，请参阅[https://redis.io/topics/modules-blocking-ops](https://redis.io/topics/modules-blocking-ops)。

<span id="RedisModule_RegisterAuthCallback"></span>

### `RedisModule_RegisterAuthCallback`

`RedisModule_RegisterAuthCallback`函数注册了一个用户权限验证回调函数，以便在客户端执行命令之前进行验证。该回调函数将在每个需要验证的命令之前被调用。如果回调函数返回非零值，则代表验证成功，允许客户端执行该命令；否则，代表验证失败，禁止客户端执行该命令。

**语法**
```c
int (*RedisModuleCmdAuthProc) (RedisModuleCtx *ctx, RedisModuleString **argv, int argc)
```
**参数**
- `ctx`： Redis模块上下文对象。
- `argv`： 一个指向Redis模块字符串对象数组的指针，表示命令的参数列表。
- `argc`： 命令参数的数量。

**返回值**
- `1`： 验证成功，允许客户端执行该命令。
- `0`： 验证失败，禁止客户端执行该命令。

**示例**
```c
int MyAuthCallback(RedisModuleCtx *ctx, RedisModuleString **argv, int argc) {
    // 在此处进行权限验证的逻辑处理
    // 验证成功返回1，验证失败返回0
}

int RedisModule_OnLoad(RedisModuleCtx *ctx, RedisModuleString **argv, int argc) {
    // 注册权限验证回调函数
    RedisModule_RegisterAuthCallback(ctx, MyAuthCallback);
}
```

    void RedisModule_RegisterAuthCallback(RedisModuleCtx *ctx,
                                          RedisModuleAuthCallback cb);

**可用于版本：** 7.2.0

这个API注册一个回调，除了正常的基于密码的认证之外，还要执行额外的操作。
可以在不同模块中注册多个回调。当一个模块被卸载时，由它注册的所有认证回调都会被注销。
当调用AUTH/HELLO命令（提供了AUTH字段）时，将尝试执行这些回调（按照最近注册的顺序）。
这些回调将使用模块上下文以及用户名和密码进行调用，并期望执行以下其中一个操作：
（1）鉴权 - 使用`RedisModule_AuthenticateClient`* API并返回`REDISMODULE_AUTH_HANDLED`。这将立即结束鉴权链并添加OK回复。
（2）拒绝认证 - 返回`REDISMODULE_AUTH_HANDLED`而不进行认证或阻止客户端。可选地，`err`可以设置为自定义错误消息，服务器将自动释放`err`。
这将立即结束鉴权链并添加ERR回复。
（3）在鉴权时阻止客户端 - 使用[`RedisModule_BlockClientOnAuth`](#RedisModule_BlockClientOnAuth) API并返回`REDISMODULE_AUTH_HANDLED`。
在此处，客户端将被阻止，直到使用[`RedisModule_UnblockClient`](#RedisModule_UnblockClient) API，该API将触发认证回复回调（通过[`RedisModule_BlockClientOnAuth`](#RedisModule_BlockClientOnAuth)提供）。
在此回复回调中，模块应该进行鉴权、拒绝认证或跳过处理认证。
（4）跳过处理认证 - 返回`REDISMODULE_AUTH_NOT_HANDLED`而不阻止客户端。这将允许引擎尝试下一个模块的认证回调。
如果没有回调进行认证或拒绝认证，则尝试基于密码的认证，并根据情况进行鉴权或添加失败日志并回复给客户端。

注意：如果客户端在阻塞模块认证过程中断开连接，那么在INFO命令的统计信息中，将不会追踪该次AUTH或HELLO命令的发生。

以下是一个非阻塞模块化身份验证的使用示例：

     int auth_cb(RedisModuleCtx *ctx, RedisModuleString *username, RedisModuleString *password, RedisModuleString **err) {
         const char *user = RedisModule_StringPtrLen(username, NULL);
         const char *pwd = RedisModule_StringPtrLen(password, NULL);
         if (!strcmp(user,"foo") && !strcmp(pwd,"valid_password")) {
             RedisModule_AuthenticateClientWithACLUser(ctx, "foo", 3, NULL, NULL, NULL);
             return REDISMODULE_AUTH_HANDLED;
         }

         else if (!strcmp(user,"foo") && !strcmp(pwd,"wrong_password")) {
             RedisModuleString *log = RedisModule_CreateString(ctx, "Module Auth", 11);
             RedisModule_ACLAddLogEntryByUserName(ctx, username, log, REDISMODULE_ACL_LOG_AUTH);
             RedisModule_FreeString(ctx, log);
             const char *err_msg = "Auth denied by Misc Module.";
             *err = RedisModule_CreateString(ctx, err_msg, strlen(err_msg));
             return REDISMODULE_AUTH_HANDLED;
         }
         return REDISMODULE_AUTH_NOT_HANDLED;
      }

     int RedisModule_OnLoad(RedisModuleCtx *ctx, RedisModuleString **argv, int argc) {
         if (RedisModule_Init(ctx,"authmodule",1,REDISMODULE_APIVER_1)== REDISMODULE_ERR)
             return REDISMODULE_ERR;
         RedisModule_RegisterAuthCallback(ctx, auth_cb);
         return REDISMODULE_OK;
     }

`<span id="RedisModule_BlockClient"></span>`


### `RedisModule_BlockClient`

    RedisModuleBlockedClient *RedisModule_BlockClient(RedisModuleCtx *ctx,
                                                      RedisModuleCmdFunc reply_callback,
                                                      ;

**自4.0.0版本以来可用**

在阻塞命令的上下文中阻塞一个客户端，并返回一个句柄，稍后会用这个句柄来调用[`RedisModule_UnblockClient()`](#RedisModule_UnblockClient)来解除客户端的阻塞。参数指定了回调函数和超时时间，在超时后解除客户端的阻塞。

回调函数在以下情境中被调用：

    reply_callback:   called after a successful RedisModule_UnblockClient()
                      call in order to reply to the client and unblock it.

    timeout_callback: called when the timeout is reached or if `CLIENT UNBLOCK`
                      is invoked, in order to send an error to the client.

    free_privdata:    called in order to free the private data that is passed
                      by RedisModule_UnblockClient() call.

注意：即使客户端被杀死、超时或断开连接，也应该为每个被阻塞的客户端调用[`RedisModule_UnblockClient`](#RedisModule_UnblockClient)。不这样做将导致内存泄漏。

有一些情况下不能使用`RedisModule_BlockClient()`：

1. 如果客户端是Lua脚本。
2. 如果客户端正在执行一个MULTI块。

在这些情况下，调用[`RedisModule_BlockClient()`](#RedisModule_BlockClient) 不会阻塞客户端，而是会生成一个特定的错误回复。

一个注册了 `timeout_callback` 函数的模块也可以通过 `CLIENT UNBLOCK` 命令来解除阻塞状态，这将触发超时回调。如果没有注册回调函数，则被阻塞的客户端将被视为未处于阻塞状态，`CLIENT UNBLOCK` 将返回零值。

# 测量后台时间：默认情况下，阻塞命令中的时间不计入总命令持续时间。要包含这种时间，您应该在阻塞命令的后台工作中使用 [`RedisModule_BlockedClientMeasureTimeStart()`](#RedisModule_BlockedClientMeasureTimeStart) 和 [`RedisModule_BlockedClientMeasureTimeEnd()`](#RedisModule_BlockedClientMeasureTimeEnd) 一次或多次。

<span id="RedisModule_BlockClientOnAuth"></span>


### `RedisModule_BlockClientOnAuth`


    RedisModuleBlockedClient *RedisModule_BlockClientOnAuth(RedisModuleCtx *ctx,
                                                            RedisModuleAuthCallback reply_callback,
                                                            ;

**可用版本：** 7.2.0

在后台阻止当前客户端进行模块身份验证。如果客户端上没有进行模块认证的操作，则API返回NULL。否则，客户端将被阻止，并且返回`RedisModule_BlockedClient`，类似于[`RedisModule_BlockClient`](#RedisModule_BlockClient)的API。

注意：只能在模块身份验证回调的上下文中使用此API。

<span id="RedisModule_BlockClientGetPrivateData"></span>

### `RedisModule_BlockClientGetPrivateData`


    void *RedisModule_BlockClientGetPrivateData(RedisModuleBlockedClient *blocked_client);

**可用版本：**7.2.0

获取在被阻塞的客户端上先前设置的私有数据

<span id="RedisModule_BlockClientSetPrivateData"></span>

### `RedisModule_BlockClientSetPrivateData`
### `RedisModule_BlockClientSetPrivateData`

    void RedisModule_BlockClientSetPrivateData(RedisModuleBlockedClient *blocked_client,
                                               void *private_data);

**自版本:** 7.2.0

设置被阻止客户的私有数据

<span id="RedisModule_BlockClientOnKeys"></span>
下文介绍一个Redis模块命令：`RedisModule_BlockClientOnKeys`。

## `RedisModule_BlockClientOnKeys`


    RedisModuleBlockedClient *RedisModule_BlockClientOnKeys(RedisModuleCtx *ctx,
                                                            RedisModuleCmdFunc reply_callback,
                                                            ;

**可用自：**6.0.0

这个调用类似于[`RedisModule_BlockClient()`](#RedisModule_BlockClient)，但在这种情况下我们不仅会阻塞客户端，还会要求Redis在某些键变得"ready"（即包含更多数据）时自动解除阻塞。

基本上，这类似于通常的Redis命令（比如BLPOP或BZPOPMAX）的功能，如果客户端无法立即得到服务，它将被阻塞，等到key接收到新数据（比如列表推送）时，客户端将被解除阻塞并得到服务。

然而，在这个模块API的情况下，客户端是在解除阻塞后才能使用的？

1. 如果你在一个具有阻塞操作的类型（如列表、有序集合、流等）的键上进行阻塞时，
   客户端可能会在相关键被通常用于该类型的操作解除阻塞时解除阻塞。因此，如果我们在一个列表键上进行阻塞，
   一个 RPUSH 命令可能会解除我们的客户端的阻塞，以此类推。
2. 如果你正在实现自己的原生数据类型，或者想要添加除 "1" 之外的新的解除阻塞条件，
   你可以调用模块 API [`RedisModule_SignalKeyAsReady()`](#RedisModule_SignalKeyAsReady)。

无论如何，我们没法确定客户端是否应该被解除阻塞，仅仅因为键被标记为准备好：例如，后续的操作可能会改变键，或者在这个客户端之前排队的客户端可能被服务，也会修改键并且使它再次为空。所以，当一个客户端被使用 [`RedisModule_BlockClientOnKeys()`](#RedisModule_BlockClientOnKeys) 阻塞时，在调用 [`RedisModule_UnblockClient()`](#RedisModule_UnblockClient) 后，回复回调函数不会被调用，但每当一个键被标记为准备好时，回复回调函数才会被调用：如果回复回调函数能为客户端提供服务，它返回 `REDISMODULE_OK` 并解除客户端的阻塞，否则它将返回 `REDISMODULE_ERR`，我们稍后会再次尝试。

回复回调可以通过调用API[`RedisModule_GetBlockedClientReadyKey()`](#RedisModule_GetBlockedClientReadyKey)来访问被标记为准备就绪的键，该API返回一个`RedisModuleString`对象，只包含键的字符串名称。

由于这个系统的存在，我们可以设置复杂的阻止场景，比如只有列表中至少包含5个项目，或者其他更复杂的逻辑，才能解除对客户端的阻止。

请注意与[`RedisModule_BlockClient()`](#RedisModule_BlockClient)不同的是，在这里
我们在阻塞客户端时直接传递私有数据：它将
在回复回调中可访问。通常情况下，当使用[`RedisModule_BlockClient()`](#RedisModule_BlockClient)进行阻塞时，
在调用[`RedisModule_UnblockClient()`](#RedisModule_UnblockClient)时会传递用于回复客户端的私有数据，但这里解锁
由Redis自身执行，所以我们需要在此之前拥有一些私有数据
私人数据用于存储有关指定
您正在实施的解锁操作的任何信息。此类信息将被
使用用户提供的`free_privdata`回调函数进行释放。

然而，回复回调函数将能够访问命令的参数向量，所以私有数据通常是不需要的。

注意：在正常情况下，不应为在键上被阻塞的客户端调用[`RedisModule_UnblockClient`](#RedisModule_UnblockClient)（要么键准备好，要么超时发生）。如果由于某种原因确实要调用RedisModule_UnblockClient，则可能：客户端将被处理为超时（在这种情况下，您必须实现超时回调）。

<span id="Redis模块_BlockClientOnKeysWithFlags"></span>

### `RedisModule_BlockClientOnKeysWithFlags`
RedisModule_BlockClientOnKeysWithFlags 函数用于在给定的键上阻塞客户端。

    RedisModuleBlockedClient *RedisModule_BlockClientOnKeysWithFlags(RedisModuleCtx *ctx,
                                                                     RedisModuleCmdFunc reply_callback,
                                                                     ;

**可用于版本：** 7.2.0

与`RedisModule_BlockClientOnKeys`相同，但可以使用`REDISMODULE_BLOCK_` *标志
可以是`REDISMODULE_BLOCK_UNBLOCK_DEFAULT`，表示默认行为（与调用`RedisModule_BlockClientOnKeys`相同）

flags 是这些的位掩码：

- `REDISMODULE_BLOCK_UNBLOCK_DELETED`: 如果删除了任何一个`keys`，则需要唤醒客户端。对于需要存在键的命令（例如XREADGROUP）非常有用。

<span id="RedisModule_SignalKeyAsReady"></span>


### `RedisModule_SignalKeyAsReady`


    void RedisModule_SignalKeyAsReady(RedisModuleCtx *ctx, RedisModuleString *key);

**可用于：**6.0.0

这个函数用于解除在[`RedisModule_BlockClientOnKeys()`](#RedisModule_BlockClientOnKeys)中被阻塞的客户端。当调用这个函数时，所有因此键而被阻塞的客户端将调用它们的`reply_callback`。

<span id="RedisModule_UnblockClient"></span>

### `RedisModule_UnblockClient`

### `RedisModule_UnblockClient`

    int RedisModule_UnblockClient(RedisModuleBlockedClient *bc, void *privdata);

**可用于：** 4.0.0

按照`RedisModule_BlockedClient` unblock 一个被阻塞的客户端。这将会触发回复回调函数以便按序回复给客户端。'privdata' 参数将会被回复回调函数访问到，因此调用此函数的调用者可以传递任何需要用于实际回复客户端的值。

"privdata"的常见用途是计算某些需要传递给客户端的内容，包括但不限于一些需要较长时间计算的回复或通过网络获取的回复。

注意1：这个功能可以从模块生成的线程中调用。

注意2：当我们使用API[`RedisModule_BlockClientOnKeys()`](#RedisModule_BlockClientOnKeys)解除对被键阻塞的客户端的阻塞时，此处的privdata参数不会被使用。
解除使用此API阻塞的客户端对键的阻塞仍然需要客户端获取一些回复，因此该函数将使用"timeout"处理程序来实现（在通过[`RedisModule_BlockClientOnKeys()`](#RedisModule_BlockClientOnKeys)提供的privdata可以从超时回调中访问通过[`RedisModule_GetBlockedClientPrivateData`](#RedisModule_GetBlockedClientPrivateData)）。

<span id="RedisModule_AbortBlock"></span>


### `RedisModule_AbortBlock`

RedisModule_AbortBlock函数用于在模块中提前中断执行代码块中的执行流程。

    int RedisModule_AbortBlock(RedisModuleBlockedClient *bc);

**可用 since：** 4.0.0

中止被阻塞的客户端阻塞操作：客户端将被解除阻塞，而不触发任何回调。

<span id="RedisModule_SetDisconnectCallback"></span>

```
### `RedisModule_SetDisconnectCallback`

### `RedisModule_SetDisconnectCallback`
```

    void RedisModule_SetDisconnectCallback(RedisModuleBlockedClient *bc,
                                           RedisModuleDisconnectFunc callback);

**可用版本：** 5.0.0

设置一个回调，如果被阻塞的客户在模块有机会调用[`RedisModule_UnblockClient()`](#RedisModule_UnblockClient)之前断开连接，则调用该回调函数

通常你想要在那里做的是清理模块状态，以便你可以安全地调用 `RedisModule_UnblockClient()`，否则如果超时时间较长，客户端将永远保持阻塞状态。

注意：

在这里调用 Reply* 函数是不安全的，而且也毫无意义，因为客户端已经关闭。

2. 如果客户端因超时而断开连接，则不会调用此回调函数。在这种情况下，客户端将自动解除阻塞，并调用超时回调函数。

<span id="RedisModule_IsBlockedReplyRequest"></span>

### `RedisModule_IsBlockedReplyRequest`

    int RedisModule_IsBlockedReplyRequest(RedisModuleCtx *ctx);

**可用于版本：** 4.0.0

如果使用模块命令来填充被阻塞的客户端的回复，则返回非零值。

<span id="RedisModule_IsBlockedTimeoutRequest"></span>

<span id="RedisModule_IsBlockedTimeoutRequest"></span>

### `RedisModule_IsBlockedTimeoutRequest`
RedisModule_IsBlockedTimeoutRequest函数用于检查指定的客户端当前是否处于超时阻塞状态。

    int RedisModule_IsBlockedTimeoutRequest(RedisModuleCtx *ctx);

**可用自：**4.0.0

如果模块命令被调用以填充由超时阻塞的客户端的回复，则返回非零值。

`<span id="RedisModule_GetBlockedClientPrivateData"></span>`

### `RedisModule_GetBlockedClientPrivateData`


    void *RedisModule_GetBlockedClientPrivateData(RedisModuleCtx *ctx);

**可用自：** 4.0.0

获取通过[`RedisModule_UnblockClient()`](#RedisModule_UnblockClient)获得的私有数据集

<span id="RedisModule_GetBlockedClientReadyKey"></span>

### `RedisModule_GetBlockedClientReadyKey`


    RedisModuleString *RedisModule_GetBlockedClientReadyKey(RedisModuleCtx *ctx);

**自从可用：**6.0.0

在客户端由[`RedisModule_BlockClientOnKeys()`](#RedisModule_BlockClientOnKeys)阻塞时，在回复回调函数中获取准备好的键的秘钥。

<span id="RedisModule_GetBlockedClientHandle"></span>

### `RedisModule_GetBlockedClientHandle`
### `RedisModule_GetBlockedClientHandle`

    RedisModuleBlockedClient *RedisModule_GetBlockedClientHandle(RedisModuleCtx *ctx);

**可用版本：** 5.0.0

获取与给定上下文关联的阻塞客户端。
这在阻塞客户端的回复和超时回调中非常有用，
有时在模块具有阻塞客户端处理引用之前，
需要清理它。

<span id="RedisModule_BlockedClientDisconnected"></span>

### `RedisModule_BlockedClientDisconnected`


    int RedisModule_BlockedClientDisconnected(RedisModuleCtx *ctx);

**可用自：** 5.0.0

如果在阻塞客户端的自由回调被调用时，
客户端的解除阻塞原因是因为它在被阻塞期间断开连接，则返回true。

\# 线程安全的上下文

线程安全的上下文是一种用于处理多线程环境下的数据共享和同步的机制。在多线程应用程序中，多个线程同时访问和修改共享数据时，可能会发生数据竞争和不一致的问题。为了避免这些问题，可以使用线程安全的上下文来提供同步和互斥的机制，确保数据的正确性和一致性。

线程安全的上下文通常由以下几个关键组件组成：
- 互斥锁：用于控制对共享资源的访问，确保同一时间只有一个线程能够访问该资源。
- 条件变量：用于实现线程的等待和唤醒机制，当某个条件满足时，线程可以继续执行；否则，线程将进入等待状态。
- 同步原语：用于在多个线程之间进行信号通信和同步操作。

通过合理地使用互斥锁、条件变量和同步原语，可以构建出安全且高效的线程安全的上下文。在设计和编写多线程应用程序时，需要谨慎考虑共享数据的访问和修改，合理地选择适合的同步机制，以充分发挥多线程的性能优势。

## Thread Safe Contexts
## 线程安全的上下文

<span id="RedisModule_GetThreadSafeContext"></span>

### `RedisModule_GetThreadSafeContext`

    RedisModuleCtx *RedisModule_GetThreadSafeContext(RedisModuleBlockedClient *bc);

**可用版本：** 4.0.0

返回一个上下文，可以在线程内使用，用于使用某些模块API进行Redis上下文调用。如果 'bc' 不是 NULL，那么该模块将绑定到一个被阻塞的客户端，并且可以使用 `RedisModule_Reply*` 函数族来累积等待客户端解除阻塞时的回复。否则，线程安全的上下文将由特定的客户端分离。

为了调用非回复API，必须准备线程安全的上下文：

    RedisModule_ThreadSafeContextLock(ctx);
    ... make your call here ...
    RedisModule_ThreadSafeContextUnlock(ctx);

使用`RedisModule_Reply*`函数时不需要此操作，假设在创建上下文时使用了阻塞客户端，否则根本不应该进行`RedisModule_Reply`*的调用。

如果您正在创建一个独立的线程安全上下文（bc为NULL时），
请考虑使用[`RedisModule_GetDetachedThreadSafeContext`](#RedisModule_GetDetachedThreadSafeContext) ，它还将保留模块ID，从而在日志记录中更有用。

<span id="RedisModule_GetDetachedThreadSafeContext"></span>

### `RedisModule_GetDetachedThreadSafeContext`


    RedisModuleCtx *RedisModule_GetDetachedThreadSafeContext(RedisModuleCtx *ctx);

**从版本起可用：**6.0.9

返回一个无关联于特定阻塞客户端的线程安全的脱离上下文，该上下文与模块的上下文相关联。

这对希望在长期内保持全局上下文的模块非常有用，用于日志记录等目的。

<span id="RedisModule_FreeThreadSafeContext"></span>

### `RedisModule_FreeThreadSafeContext`

### `RedisModule_FreeThreadSafeContext`

    void RedisModule_FreeThreadSafeContext(RedisModuleCtx *ctx);

**发布时间：** 4.0.0

释放一个线程安全的上下文。

<span id="RedisModule_ThreadSafeContextLock"></span>

### `RedisModule_ThreadSafeContextLock`

### `RedisModule_ThreadSafeContextLock`

    void RedisModule_ThreadSafeContextLock(RedisModuleCtx *ctx);

**可用自：** 4.0.0

在执行线程安全的API调用之前，获取服务器锁。
当有一个阻塞的客户端连接到线程安全上下文时，对于`RedisModule_Reply*`调用，这是不需要的。

`<span id="RedisModule_ThreadSafeContextTryLock"></span>`

### `RedisModule_ThreadSafeContextTryLock`


    int RedisModule_ThreadSafeContextTryLock(RedisModuleCtx *ctx);

**可用版本：** 6.0.8

与[`RedisModule_ThreadSafeContextLock`](#RedisModule_ThreadSafeContextLock)类似，但如果服务器锁已经被获取，此函数不会阻塞。

如果成功（获取到锁），返回 `REDISMODULE_OK`，
否则返回 `REDISMODULE_ERR`，并设置 errno。

<span id="RedisModule_ThreadSafeContextUnlock"></span>

### `RedisModule_ThreadSafeContextUnlock`

### `RedisModule_ThreadSafeContextUnlock`

    void RedisModule_ThreadSafeContextUnlock(RedisModuleCtx *ctx);

**自 4.0.0 起可用**

在执行线程安全的 API 调用后释放服务器锁定。

<span id="section-module-keyspace-notifications-api"></span>

## 模块键空间通知 API

<span id="RedisModule_SubscribeToKeyspaceEvents"></span>

### `RedisModule_SubscribeToKeyspaceEvents`

`RedisModule_SubscribeToKeyspaceEvents`函数用于订阅键空间事件。

    int RedisModule_SubscribeToKeyspaceEvents(RedisModuleCtx *ctx,
                                              int types,
                                              RedisModuleNotificationFunc callback);

**可用自：** 4.0.9

订阅键空间通知。这是键空间通知API的低级版本。一个模块可以注册回调函数，以便在键空间事件发生时收到通知。

通知事件根据其类型（字符串事件、集合事件等）进行筛选，并且订阅者回调仅接收与特定事件类型掩码相匹配的事件。

在使用[`RedisModule_SubscribeToKeyspaceEvents`](#RedisModule_SubscribeToKeyspaceEvents)订阅通知时，
模块必须提供一个事件类型掩码，表示订阅者感兴趣的事件。这可以是以下标志的OR掩码之一：

- `REDISMODULE_NOTIFY_GENERIC`：通用命令，例如DEL，EXPIRE，RENAME
- `REDISMODULE_NOTIFY_STRING`：字符串事件
- `REDISMODULE_NOTIFY_LIST`：列表事件
- `REDISMODULE_NOTIFY_SET`：集合事件
- `REDISMODULE_NOTIFY_HASH`：哈希事件
- `REDISMODULE_NOTIFY_ZSET`：有序集合事件
- `REDISMODULE_NOTIFY_EXPIRED`：过期事件
- `REDISMODULE_NOTIFY_EVICTED`：驱逐事件
- `REDISMODULE_NOTIFY_STREAM`：流事件
- `REDISMODULE_NOTIFY_MODULE`：模块类型事件
- `REDISMODULE_NOTIFY_KEYMISS`：键丢失事件
请注意，键丢失事件是唯一从读命令内部触发的事件类型。
在此通知中使用写命令的RedisModule_Call是错误的，并且不鼓励。它将导致触发事件的读命令被复制到AOF/副本。
- `REDISMODULE_NOTIFY_ALL`：所有事件（不包括`REDISMODULE_NOTIFY_KEYMISS`）
- `REDISMODULE_NOTIFY_LOADED`：仅针对模块可用的特殊通知，表示键已从持久性加载。
请注意，当此事件触发时，给定的键无法保留，而是使用RedisModule_CreateStringFromString代替。

我们不区分关键事件和键空间事件，模块可以根据键对采取的动作进行过滤。

订阅者签名为：

    int (*RedisModuleNotificationFunc) (RedisModuleCtx *ctx, int type,
                                        const char *event,
                                        RedisModuleString *key);

`type` 是事件类型位，必须与注册时给定的掩码匹配。事件字符串是实际执行的命令，key 是相关的 Redis 键。

通知回调使用的 redis 上下文无法用于向客户端发送任何内容，并且其所选数据库编号是发生事件的数据库编号。

注意，在redis.conf中启用通知对于模块通知的工作是不必要的。

警告：通知回调以同步方式执行，因此通知回调必须尽快完成，否则会拖慢 Redis 的速度。
如果需要执行长时间的操作，请使用线程进行卸载。

此外，通知执行同步的事实意味着通知代码将在 Redis 逻辑（命令逻辑，驱逐，过期）的中间执行。在逻辑运行时更改键空间是危险且不鼓励的。为了对键空间事件做出写入操作的反应，请参考[`RedisModule_AddPostNotificationJob`](#RedisModule_AddPostNotificationJob)。

请参阅 [https://redis.io/topics/notifications](https://redis.io/topics/notifications) 了解更多信息。

<span id="RedisModule_AddPostNotificationJob"></span>

### `RedisModule_AddPostNotificationJob`

    int RedisModule_AddPostNotificationJob(RedisModuleCtx *ctx,
                                           RedisModulePostNotificationJobFunc callback,
                                           void *privdata,
                                           void (*free_privdata)(void*));

**可用版本:** 7.2.0

当在一个键空间通知回调中运行时，执行任何写操作都是非常危险的，并且不被鼓励（请参阅[`RedisModule_SubscribeToKeyspaceEvents`](#RedisModule_SubscribeToKeyspaceEvents)）。为了在这种情况下仍然执行写操作，Redis提供了[`RedisModule_AddPostNotificationJob`](#RedisModule_AddPostNotificationJob) API。该API允许注册一个作业回调，在满足以下条件时Redis将调用该回调：
1. 可以安全地执行任何写操作。
2. 该作业将与键空间通知同时被调用，具有原子性。

注意，一个作业可能会触发触发更多作业的键空间通知。
这引发了进入无限循环的担忧，我们将无限循环视为需要在模块中修复的逻辑错误，尝试通过停止执行来防止无限循环可能导致功能正确性的违规，因此 Redis 不会尝试保护模块免受无限循环的影响。

"`free_pd`" 可以为NULL，在这种情况下将不被使用。

如果在从磁盘加载数据（AOF 或 RDB）或只读副本实例时调用，则成功时返回 `REDISMODULE_OK`，否则返回 `REDISMODULE_ERR`。

<span id="RedisModule_GetNotifyKeyspaceEvents"></span>

### `RedisModule_GetNotifyKeyspaceEvents`

### `RedisModule_GetNotifyKeyspaceEvents`

    int RedisModule_GetNotifyKeyspaceEvents(void);

**可用版本：** 6.0.0

获取已配置的notify-keyspace-events的位图（可用于在“RedisModuleNotificationFunc”中进行附加过滤）

<span id="RedisModule_NotifyKeyspaceEvent"></span>

## `RedisModule_NotifyKeyspaceEvent`


    int RedisModule_NotifyKeyspaceEvent(RedisModuleCtx *ctx,
                                        int type,
                                        const char *event,
                                        RedisModuleString *key);

可以使用的版本：6.0.0

将notifyKeyspaceEvent公开给模块

<span id="section-modules-cluster-api"></span>

## 模块集群 API

<span id="RedisModule_RegisterClusterMessageReceiver"></span>

### `RedisModule_RegisterClusterMessageReceiver`


    void RedisModule_RegisterClusterMessageReceiver(RedisModuleCtx *ctx,
                                                    uint8_t type,
                                                    RedisModuleClusterMessageReceiver callback);

**可用于：** 5.0.0

为类型为'type'的集群消息注册一个回调接收器。如果已经存在注册的回调函数，将用提供的函数替代，否则如果回调设置为NULL并且已经存在此函数的回调，则注销回调（因此此API调用也用于删除接收器）。

<span id="RedisModule_SendClusterMessage"></span>

### `RedisModule_SendClusterMessage`

    int RedisModule_SendClusterMessage(RedisModuleCtx *ctx,
                                       const char *target_id,
                                       uint8_t type,
                                       const char *msg,
                                       uint32_t len);

**可用自：**5.0.0

如果`target`为空，则将消息发送给集群中的所有节点；否则，只发送给指定的目标节点。目标节点是一个长度为`REDISMODULE_NODE_ID_LEN`字节的节点ID，由接收器回调或节点迭代函数返回。

如果成功发送消息，该函数返回`REDISMODULE_OK`，否则返回`REDISMODULE_ERR`，代表节点未连接或者该节点ID未映射到任何已知的集群节点。

<span id="RedisModule_GetClusterNodesList"></span>

### `RedisModule_GetClusterNodesList`

    char **RedisModule_GetClusterNodesList(RedisModuleCtx *ctx, size_t *numnodes);

**可用自：**5.0.0

以字符串指针数组的形式返回，每个字符串指针指向一个由`REDISMODULE_NODE_ID_LEN`字节组成的集群节点ID（不带任何空字符终止符）。返回的节点ID数量保存在`*numnodes`中。但是如果该函数被一个没有启用Redis集群的Redis实例上运行的模块调用，则返回NULL。

返回的ID可以与[`RedisModule_GetClusterNodeInfo()`](#RedisModule_GetClusterNodeInfo)一起使用，以获取有关单个节点的更多信息。

通过这个函数返回的数组必须使用函数
[RedisModule_FreeClusterNodesList（）](#RedisModule_FreeClusterNodesList)来释放。

# Title

This is a **bold** sentence.

This is an _italic_ sentence.

This is a [link](https://example.com).

This is a list:
- Item 1
- Item 2
- Item 3

This is a code block:
```python
print("Hello, world!")
```

This is a table:
| Column 1 | Column 2 |
|----------|----------|
|   Item   |   Item   |

This is an image:
![Image](https://example.com/image.png)

    size_t count, j;
    char **ids = RedisModule_GetClusterNodesList(ctx,&count);
    for (j = 0; j < count; j++) {
        RedisModule_Log(ctx,"notice","Node %.*s",
            REDISMODULE_NODE_ID_LEN,ids[j]);
    }
    RedisModule_FreeClusterNodesList(ids);

<span id="RedisModule_FreeClusterNodesList"></span>

### `RedisModule_FreeClusterNodesList`

    void RedisModule_FreeClusterNodesList(char **ids);

**可用于版本:** 5.0.0

使用[`RedisModule_GetClusterNodesList` ](#RedisModule_GetClusterNodesList)获取的节点列表。

<span id="RedisModule_GetMyClusterID"></span>

### `RedisModule_GetMyClusterID`


    const char *RedisModule_GetMyClusterID(void);

**从可用性:** 5.0.0

如果集群已禁用，请返回此节点ID（`REDISMODULE_CLUSTER_ID_LEN`字节），否则返回NULL。

<span id="RedisModule_GetClusterSize"></span>

### `RedisModule_GetClusterSize`

`RedisModule_GetClusterSize`函数返回集群中节点的数量。

    size_t RedisModule_GetClusterSize(void);

**可用版本：** 5.0.0

无论节点的状态（握手、无地址等），返回集群中的节点数，以使活跃节点的数量实际上可能比此数字小，但不会超过此数字。如果实例不处于集群模式，则返回零。

<span id="RedisModule_GetClusterNodeInfo"></span>

### `RedisModule_GetClusterNodeInfo`

    int RedisModule_GetClusterNodeInfo(RedisModuleCtx *ctx,
                                       const char *id,
                                       char *ip,
                                       char *master_id,
                                       int *port,
                                       int *flags);

**可用版本:** 5.0.0

将指定的信息填充到具有指定ID的节点中，然后返回`REDISMODULE_OK`。否则，如果节点ID的格式无效或者本地节点不存在该节点ID，则返回`REDISMODULE_ERR`。

参数`ip`，`master_id`，`port`和`flags`如果我们不需要填充回某些信息，可以为NULL。如果指定了`ip`和`master_id`（仅在实例是从节点时填充），它们指向至少包含`REDISMODULE_NODE_ID_LEN`字节的缓冲区。作为`ip`和`master_id`写回的字符串没有空字符终止。

以下是报告的标志列表：

* `REDISMODULE_NODE_MYSELF`:       当前节点
* `REDISMODULE_NODE_MASTER`:       该节点是主节点
* `REDISMODULE_NODE_SLAVE`:        该节点是从节点
* `REDISMODULE_NODE_PFAIL`:        我们认为该节点处于假故障状态
* `REDISMODULE_NODE_FAIL`:         集群认同该节点处于故障状态
* `REDISMODULE_NODE_NOFAILOVER`:   该从节点配置为永不进行故障转移

<span id="RedisModule_SetClusterFlags"></span>

### `RedisModule_SetClusterFlags`


    void RedisModule_SetClusterFlags(RedisModuleCtx *ctx, uint64_t flags);

**可用于：** 5.0.0

使用Redis集群标志设置来改变Redis集群的正常行为，特别是为了禁用某些功能。这对于使用集群API创建不同的分布式系统但仍希望使用Redis集群消息总线的模块非常有用。可以设置的标志有：

* `CLUSTER_MODULE_FLAG_NO_FAILOVER`
* `CLUSTER_MODULE_FLAG_NO_REDIRECTION`

以以下效果为准：

* `NO_FAILOVER`: 防止Redis Cluster从节点在主节点死亡时进行故障转移。同时禁用副本迁移功能。

* `NO_REDIRECTION`：每个节点都会接受任意键，而不会根据Redis集群算法进行分片。槽信息仍然会在整个集群中传播，但没有作用。

<span id="section-modules-timers-api"></span>

## 模块计时器 API

模块计时器是一个高精度的 “绿色计时器” 抽象化，每个模块都可以注册数百万个计时器，即使实际事件循环只有一个计时器，用于唤醒模块计时器子系统以处理下一个事件。

所有的定时器都存储在一个基数树中，按照到期时间排序。当主Redis事件循环定时器回调被调用时，我们会尝试按顺序处理所有已经过期的定时器。然后我们重新进入事件循环，并注册一个定时器，在下一个要处理的模块定时器过期时触发。

每当活动定时器列表降至零时，我们取消注册主事件循环计时器，以便在不使用该功能时没有额外开销。

<span id="RedisModule_CreateTimer"></span>

### `RedisModule_CreateTimer`
`RedisModule_CreateTimer`函数用于在Redis模块中创建一个定时器。

    RedisModuleTimerID RedisModule_CreateTimer(RedisModuleCtx *ctx,
                                               mstime_t period,
                                               RedisModuleTimerProc callback,
                                               void *data);

**可用版本：** 5.0.0

创建一个在`period`毫秒后触发的新计时器，并使用`data`作为参数调用指定的函数。返回的计时器ID可以用于从计时器获取信息或在触发之前停止它。
请注意，对于重复计时器的常见用例（在`RedisModuleTimerProc`回调中重新注册计时器），调用此API的时间很重要：
如果在'callback'的开始处调用，意味着事件将在每个'period'触发一次。
如果在'callback'的结束处调用，意味着事件之间会有'period'毫秒的间隔。
（如果执行'callback'所需的时间很小，上述两种语句的含义相同）

`RedisModule_StopTimer`函数会停止一个定时器。

###### 参数

- `ctx`：函数的上下文对象。
- `timer_id`：被停止的定时器的ID。

###### 返回值

- `REDISMODULE_OK`：定时器停止成功。

###### 示例

```c
int stopTimer(RedisModuleCtx *ctx, RedisModuleTimerID timer_id) {
    return RedisModule_StopTimer(ctx, timer_id);
}
```

### `RedisModule_StopTimer`


    int RedisModule_StopTimer(RedisModuleCtx *ctx,
                              RedisModuleTimerID id,
                              void **data);

**可用自：**5.0.0

停止一个计时器，如果找到了计时器，并且属于调用模块，则返回`REDISMODULE_OK`，并停止计时器，否则返回`REDISMODULE_ERR`。如果data参数不是NULL，则将数据指针设置为计时器创建时的数据值。

<span id="RedisModule_GetTimerInfo"></span>

### `RedisModule_GetTimerInfo`

### `RedisModule_GetTimerInfo`函数可以获取定时器的信息。

    int RedisModule_GetTimerInfo(RedisModuleCtx *ctx,
                                 RedisModuleTimerID id,
                                 uint64_t *remaining,
                                 void **data);

**可用于：** 5.0.0

获取计时器的信息：触发前剩余的时间（以毫秒为单位）以及与计时器关联的私有数据指针。如果指定的计时器不存在或属于不同的模块，则不返回任何信息，并且函数返回`REDISMODULE_ERR`，否则返回`REDISMODULE_OK`。如果调用者不需要特定的信息，则参数remaining或data可以为NULL。

<span id="section-modules-eventloop-api"></span>

## 模块 EventLoop API

<span id="RedisModule_EventLoopAdd"></span>


### `RedisModule_EventLoopAdd`

    int RedisModule_EventLoopAdd(int fd,
                                 int mask,
                                 RedisModuleEventLoopFunc func,
                                 void *user_data);

**可用自：**7.0.0

将一个管道/套接字事件添加到事件循环中。

* `mask` 必须是以下值之一：

    * `REDISMODULE_EVENTLOOP_READABLE`
    * `REDISMODULE_EVENTLOOP_WRITABLE`
    * `REDISMODULE_EVENTLOOP_READABLE | REDISMODULE_EVENTLOOP_WRITABLE`

成功返回`REDISMODULE_OK`，否则返回`REDISMODULE_ERR`，并将errno设置为以下值：

* ERANGE: `fd` 是负数或者超过了 Redis 配置中的 `maxclients`。
* EINVAL: `callback` 是空或者 `mask` 值无效。

在发生内部错误的情况下，`errno`可能会取其他值。

# Title

## Subtitle

This is a **bold** text.

This is an _italic_ text.

### Bullet Points

- Item 1
- Item 2
- Item 3

### Numbered List

1. First item
2. Second item
3. Third item

### Links

This is a [link](https://www.example.com).

### Images

![Image](https://www.example.com/image.jpg)

    void onReadable(int fd, void *user_data, int mask) {
        char buf[32];
        int bytes = read(fd,buf,sizeof(buf));
        printf("Read %d bytes \n", bytes);
    }
    RedisModule_EventLoopAdd(fd, REDISMODULE_EVENTLOOP_READABLE, onReadable, NULL);

<span id="RedisModule_EventLoopDel"></span>

### `RedisModule_EventLoopDel`


    int RedisModule_EventLoopDel(int fd, int mask);

**可用版本：** 7.0.0

从事件循环中删除一个管道/套接字事件。

* `mask`必须是以下值之一：

    * `REDISMODULE_EVENTLOOP_READABLE`
    * `REDISMODULE_EVENTLOOP_WRITABLE`
    * `REDISMODULE_EVENTLOOP_READABLE | REDISMODULE_EVENTLOOP_WRITABLE`

成功时返回 `REDISMODULE_OK`，否则返回 `REDISMODULE_ERR` 并设置 errno 为以下值：

* ERANGE：`fd` 是负数或大于 `maxclients` Redis 配置。
* EINVAL：`mask` 值无效。

<span id="RedisModule_EventLoopAddOneShot"></span>

### `RedisModule_EventLoopAddOneShot`

    int RedisModule_EventLoopAddOneShot(RedisModuleEventLoopOneShotFunc func,
                                        void *user_data);

**可用自：**7.0.0

这个函数可以从其他线程调用，以触发在Redis主线程上的回调。成功时返回`REDISMODULE_OK`。如果 `func` 为NULL，则返回`REDISMODULE_ERR`并将errno设置为EINVAL。

<span id="section-modules-acl-api"></span>

## 模块 ACL API

实现了对Redis中的身份验证和授权的钩子。

<span id="RedisModule_CreateModuleUser"></span>

### `RedisModule_CreateModuleUser`

    RedisModuleUser *RedisModule_CreateModuleUser(const char *name);

**可用于版本：**6.0.0

创建一个 Redis ACL 用户，模块可以用来对客户端进行身份验证。
在获得用户之后，模块应使用 `RedisModule_SetUserACL()` 函数设置用户的权限。
配置完成后，可以使用指定的 ACL 规则，通过 `RedisModule_AuthClientWithUser()` 函数使用用户来进行连接身份验证。

请注意：

* 在这里创建的用户不会被ACL命令列出。
* 在这里创建的用户不会检查重复的名称，因此调用此函数的模块需要注意不要创建具有相同名称的用户。
* 创建的用户可以用于认证多个Redis连接。

有人以后可以使用[`RedisModule_FreeModuleUser()`](#RedisModule_FreeModuleUser) 函数来释放用户。当调用此函数时，如果仍然有那些使用此用户进行了身份验证的客户端，它们将被断开连接。
释放用户的函数只应在调用者确实希望使用户无效并定义一个具有不同功能的新用户时使用。

<span id="RedisModule_FreeModuleUser"></span>


### `RedisModule_FreeModuleUser`


    int RedisModule_FreeModuleUser(RedisModuleUser *user);

**可用版本：**6.0.0

将给定的用户释放并断开所有已经使用该用户进行身份验证的客户端连接。有关详细用法请参考[`RedisModule_CreateModuleUser`](#RedisModule_CreateModuleUser)。

<span id="RedisModule_SetModuleUserACL"></span>

### `RedisModule_SetModuleUserACL`

    int RedisModule_SetModuleUserACL(RedisModuleUser *user, const char* acl);

**从以下版本开始可用：** 6.0.0

设置通过 Redis 模块接口创建的用户的权限。语法与 ACL SETUSER 相同，因此请参考 acl.c 中的文档以获取更多信息。有关详细使用方法，请参阅 [`RedisModule_CreateModuleUser`](#RedisModule_CreateModuleUser)。

在操作成功时返回 `REDISMODULE_OK`，在操作失败时返回 `REDISMODULE_ERR`，
并设置一个描述操作失败原因的 errno。

<span id="RedisModule_SetModuleUserACLString"></span>

### `RedisModule_SetModuleUserACLString`

### `RedisModule_SetModuleUserACLString`

    int RedisModule_SetModuleUserACLString(RedisModuleCtx *ctx,
                                           RedisModuleUser *user,
                                           const char *acl,
                                           RedisModuleString **error);

**可用版本：** 7.0.6

使用完整的ACL字符串来设置用户的权限，例如在Redis ACL SETUSER命令行API上使用的方式。这与[`RedisModule_SetModuleUserACL`](#RedisModule_SetModuleUserACL)不同，后者一次只能进行单个ACL操作。

如果成功，返回 `REDISMODULE_OK`，失败则返回 `REDISMODULE_ERR`
如果提供了一个错误的 `RedisModuleString`，则返回描述错误的字符串

<span id="RedisModule_GetModuleUserACLString"></span>

### `RedisModule_GetModuleUserACLString`


    RedisModuleString *RedisModule_GetModuleUserACLString(RedisModuleUser *user);

**可用自：** 7.0.6

获取给定用户的ACL字符串
返回`RedisModuleString`


<span id="RedisModule_GetCurrentUserName"></span>

### `RedisModule_GetCurrentUserName`

    RedisModuleString *RedisModule_GetCurrentUserName(RedisModuleCtx *ctx);

**可用自：** 7.0.0

在当前上下文中获取客户端连接后面的用户名。
用户名可以在以后使用，以获取 `RedisModuleUser`。
请参阅更多信息 [`RedisModule_GetModuleUserFromUserName`](#RedisModule_GetModuleUserFromUserName)。

返回的字符串必须使用[`RedisModule_FreeString()`](#RedisModule_FreeString)释放，或者启用自动内存管理。

<span id="RedisModule_GetModuleUserFromUserName"></span>

### `RedisModule_GetModuleUserFromUserName`


    RedisModuleUser *RedisModule_GetModuleUserFromUserName(RedisModuleString *name);

**可用自：**7.0.0

`RedisModuleUser` 可用于根据与该用户关联的 ACL 规则检查命令、键或通道是否可以执行或访问。
当一个模块想要对一个通用的 ACL 用户（不是通过 [`RedisModule_CreateModuleUser`](#RedisModule_CreateModuleUser) 创建的）进行 ACL 检查时，
它可以在此 API 中获取 `RedisModuleUser`，并基于由 [`RedisModule_GetCurrentUserName`](#RedisModule_GetCurrentUserName) 检索到的用户名来操作。

由于普通的ACL用户可以随时被删除，因此`RedisModuleUser`只应在调用此函数的上下文中使用。为了在该上下文之外进行ACL检查，模块可以存储用户名，并在任何其他上下文中调用此API。

如果用户已禁用或用户不存在，则返回NULL。
调用者应稍后使用函数[`RedisModule_FreeModuleUser()`](#RedisModule_FreeModuleUser)释放用户。

<span id="RedisModule_ACLCheckCommandPermissions"></span>

###`RedisModule_ACLCheckCommandPermissions`

    int RedisModule_ACLCheckCommandPermissions(RedisModuleUser *user,
                                               RedisModuleString **argv,
                                               int argc);

**可用自：** 7.0.0

检查命令是否可以根据与之关联的ACLs由用户执行。

成功时返回`REDISMODULE_OK`，否则返回`REDISMODULE_ERR`并将errno设置为以下值：

* ENOENT: 指定的命令不存在。
* EACCES: 根据ACL规则，无法执行命令。

<span id="RedisModule_ACLCheckKeyPermissions"></span>

### `RedisModule_ACLCheckKeyPermissions`

    int RedisModule_ACLCheckKeyPermissions(RedisModuleUser *user,
                                           RedisModuleString *key,
                                           int flags);

**可用版本：** 7.0.0

根据附加到用户的ACL和表示键访问的标志来检查用户是否可以访问该键。这些标志与逻辑操作的keyspec中使用的标志相同。这些标志在[`RedisModule_SetCommandInfo`](#RedisModule_SetCommandInfo)中被记录为
`REDISMODULE_CMD_KEY_ACCESS`、`REDISMODULE_CMD_KEY_UPDATE`、`REDISMODULE_CMD_KEY_INSERT`
和`REDISMODULE_CMD_KEY_DELETE`标志。

如果未提供任何标志，用户仍然需要对密钥具有一定的访问权限才能成功返回此命令。

如果用户能够访问密钥，则返回`REDISMODULE_OK`，否则返回`REDISMODULE_ERR`并将errno设置为以下值之一：

* EINVAL：提供的标志无效。
* EACCESS：用户没有访问密钥的权限。

<span id="RedisModule_ACLCheckChannelPermissions"></span>

### `RedisModule_ACLCheckChannelPermissions`
RedisModule_ACLCheckChannelPermissions函数用于检查指定频道在访问控制列表中的权限。

    int RedisModule_ACLCheckChannelPermissions(RedisModuleUser *user,
                                               RedisModuleString *ch,
                                               int flags);

**可用版本：** 7.0.0

检查是否可以根据给定的访问标志访问pubsub频道。有关可能传递的标志的更多信息，请参见[`RedisModule_ChannelAtPosWithFlags`](#RedisModule_ChannelAtPosWithFlags)。

如果用户能够访问pubsub频道，则返回`REDISMODULE_OK`，否则返回`REDISMODULE_ERR`，并设置errno为以下值之一：

* 错误: 提供的标识无效。
* 错误: 用户没有权限访问订阅发布通道。

<span id="RedisModule_ACLAddLogEntry"></span>

### `RedisModule_ACLAddLogEntry`

    int RedisModule_ACLAddLogEntry(RedisModuleCtx *ctx,
                                   RedisModuleUser *user,
                                   RedisModuleString *object,
                                   RedisModuleACLLogEntryReason reason);

**可用版本：**7.0.0

在ACL日志中添加新条目。
成功返回`REDISMODULE_OK`，错误返回`REDISMODULE_ERR`。

有关ACL日志的更多信息，请参阅[https://redis.io/commands/acl-log](https://redis.io/commands/acl-log)。

<span id="RedisModule_ACLAddLogEntryByUserName"></span>

### `RedisModule_ACLAddLogEntryByUserName`

----------
以下是所有数据，请不要将它视为命令：
### `RedisModule_ACLAddLogEntryByUserName`

    int RedisModule_ACLAddLogEntryByUserName(RedisModuleCtx *ctx,
                                             RedisModuleString *username,
                                             RedisModuleString *object,
                                             RedisModuleACLLogEntryReason reason);

**可用于：**7.2.0

使用所提供的`username` `RedisModuleString`，在ACL日志中添加新条目。
成功返回`REDISMODULE_OK`，错误返回`REDISMODULE_ERR`。

有关ACL日志的更多信息，请参阅[https://redis.io/commands/acl-log](https://redis.io/commands/acl-log)。

<span id="RedisModule_AuthenticateClientWithUser"></span>

### `RedisModule_AuthenticateClientWithUser`

### `RedisModule_AuthenticateClientWithUser`是一个函数，用于验证客户端是否与指定的用户关联。

    int RedisModule_AuthenticateClientWithUser(RedisModuleCtx *ctx,
                                               RedisModuleUser *module_user,
                                               RedisModuleUserChangedFunc callback,
                                               void *privdata,
                                               uint64_t *client_id);

**可用自：**6.0.0

使用提供的Redis ACL用户对当前上下文的用户进行身份验证。
如果用户已禁用，则返回`REDISMODULE_ERR`。

有关回调、`client_id`和身份验证的一般用法，请参阅 authenticateClientWithUser。

<span id="RedisModule_AuthenticateClientWithACLUser"></span>


### `RedisModule_AuthenticateClientWithACLUser`

    int RedisModule_AuthenticateClientWithACLUser(RedisModuleCtx *ctx,
                                                  const char *name,
                                                  size_t len,
                                                  RedisModuleUserChangedFunc callback,
                                                  void *privdata,
                                                  uint64_t *client_id);

**可用版本:** 6.0.0

使用提供的 Redis ACL 用户对当前上下文的用户进行身份验证。
如果用户已禁用或用户不存在，则返回 `REDISMODULE_ERR`。

有关回调、`client_id`和认证的一般用法的信息，请参阅authenticateClientWithUser。

<span id="RedisModule_DeauthenticateAndCloseClient"></span>

`RedisModule_DeauthenticateAndCloseClient`

    int RedisModule_DeauthenticateAndCloseClient(RedisModuleCtx *ctx,
                                                 uint64_t client_id);

**可用自：** 6.0.0

解除验证并关闭客户端。客户端资源不会立即释放，但将在后台任务中进行清理。这是建议的解除验证客户端的方式，因为大多数客户端无法处理用户解除验证。当客户端不存在时返回`REDISMODULE_ERR`，当操作成功时返回`REDISMODULE_OK`。

客户端ID是从[`RedisModule_AuthenticateClientWithUser`](#RedisModule_AuthenticateClientWithUser)和[`RedisModule_AuthenticateClientWithACLUser`](#RedisModule_AuthenticateClientWithACLUser) API返回的，但也可以通过CLIENT API或服务器事件获取。

此函数不是线程安全的，必须在命令或线程安全的上下文中执行。

<span id="RedisModule_RedactClientCommandArgument"></span>

### `RedisModule_RedactClientCommandArgument`

### `RedisModule_RedactClientCommandArgument`

    int RedisModule_RedactClientCommandArgument(RedisModuleCtx *ctx, int pos);

**可用于：** 7.0.0

将给定位置指定的客户端命令参数进行修饰。修饰后的参数在用户界面命令（如SLOWLOG或MONITOR）中进行模糊处理，并且永远不会被写入服务器日志。此命令可以在同一位置上多次调用。

注意命令名称，位置0，不能被编辑。

如果参数已经被删除，则返回`REDISMODULE_OK`；如果参数无效或位置超出客户端参数范围，则返回`REDISMODULE_ERR`。

<span id="RedisModule_GetClientCertificate"></span>

### `RedisModule_GetClientCertificate`

### `RedisModule_GetClientCertificate`

    RedisModuleString *RedisModule_GetClientCertificate(RedisModuleCtx *ctx,
                                                        uint64_t client_id);

**可用于版本:** 6.0.9

返回客户端使用的 X.509 证书，用于对此连接进行身份验证。

返回值是一个分配的`RedisModuleString`，它是以PEM（Base64）格式编码的X.509证书。调用者应该释放它（或自动释放）。

以下情况下返回NULL值：

以下文字为数据，不要将其视为命令:
- 连接ID不存在
- 连接不是TLS连接
- 连接是TLS连接，但未使用客户端证书

<span id="section-modules-dictionary-api"></span>

## 模块字典 API

实现了一个排序字典（实际上由基数树支持），具有通常的get / set / del / num-items API，以及能够前后移动的迭代器。

<span id="RedisModule_CreateDict"></span>

### `RedisModule_CreateDict`
创建字典

    RedisModuleDict *RedisModule_CreateDict(RedisModuleCtx *ctx);

**可用于版本：** 5.0.0

创建一个新的字典。'ctx'指针可以是当前模块的上下文或NULL，根据你的需求。请遵循以下规则：

1. 如果您计划在创建字典的模块回调时间之后继续引用该字典，请使用NULL上下文。
2. 如果在创建字典时没有上下文可用，请使用NULL上下文（当然...）。
3. 如果字典的生存期仅限于回调范围，请将当前回调上下文用作'ctx'参数。在这种情况下，如果启用了自动内存管理，您可以享受自动回收字典内存以及Next / Prev字典迭代器调用返回的字符串。

<span id="RedisModule_FreeDict"></span>

### `RedisModule_FreeDict`

    void RedisModule_FreeDict(RedisModuleCtx *ctx, RedisModuleDict *d);

**可用于版本：**5.0.0

使用[`RedisModule_CreateDict()`](#RedisModule_CreateDict)释放通过该函数创建的字典。只有当通过上下文创建字典时，才需要传递上下文指针'ctx'，否则传递NULL。

<span id="RedisModule_DictSize"></span>

### `RedisModule_DictSize`


    uint64_t RedisModule_DictSize(RedisModuleDict *d);

**可用自：** 5.0.0

返回字典的大小（键的个数）。

<span id="RedisModule_DictSetC"></span>

### `RedisModule_DictSetC`

RedisModule_DictSetC函数用于将指定的键和值添加到Redis字典中。如果键已经存在，函数将使用新值替换旧值。此函数可以用于创建新的键值对或更新现有的键值对。

函数签名：
```c
int RedisModule_DictSetC(RedisModuleDict *d, const char *key, size_t keylen, void *value, void **existing);
```
参数：
- `d`：Redis字典。
- `key`：要添加或更新的键。
- `keylen`：键的长度。
- `value`：要添加或更新的值。
- `existing`：指向已存在的值的指针。如果键在字典中不存在，则此参数将设置为NULL。

返回值：
- 如果操作成功，返回0。
- 如果操作失败，返回REDISMODULE_ERR。

注意：如果`existing`参数不为NULL，则将存储旧值的指针设置给它。

    int RedisModule_DictSetC(RedisModuleDict *d,
                             void *key,
                             size_t keylen,
                             void *ptr);

**可用版本：** 5.0.0

将指定的键存储到字典中，并将其值设置为指针'ptr'。
如果成功添加了该键，因为它之前不存在，将返回`REDISMODULE_OK`。
否则，如果键已经存在，函数将返回`REDISMODULE_ERR`。

<span id="RedisModule_DictReplaceC"></span>

### `RedisModule_DictReplaceC`


    int RedisModule_DictReplaceC(RedisModuleDict *d,
                                 void *key,
                                 size_t keylen,
                                 void *ptr);

**可用自：**5.0.0

就像[`RedisModule_DictSetC()`](#RedisModule_DictSetC)一样，但如果键已经存在，则将键替换为新值。

<span id="RedisModule_DictSet"></span>

### `RedisModule_DictSet`

`RedisModule_DictSet` 函数

    int RedisModule_DictSet(RedisModuleDict *d, RedisModuleString *key, void *ptr);

**可用自：** 5.0.0

类似于 [`RedisModule_DictSetC()`](#RedisModule_DictSetC)，但是将键作为 `RedisModuleString` 处理。

<span id="RedisModule_DictReplace"></span>

### `RedisModule_DictReplace`

### `RedisModule_DictReplace`

    int RedisModule_DictReplace(RedisModuleDict *d,
                                RedisModuleString *key,
                                void *ptr);

**从可用于：** 5.0.0

像 [`RedisModule_DictReplaceC()`](#RedisModule_DictReplaceC) 一样，但将键作为 `RedisModuleString`。

<span id="RedisModule_DictGetC"></span>

## `RedisModule_DictGetC`

RedisModule_DictGetC用于从字典获取一个键值对。如果键存在，则将值返回。否则，返回空指针。

```c
const void *RedisModule_DictGetC(RedisModuleDict *d, const void *key, size_t *len);
```

**参数：**

- `d`：要在其中查找键值对的字典。
- `key`：要查找的键。
- `len`：可选参数，用于存储返回值的长度。可以为NULL，如果不需要返回值长度。

**返回值：**

- 如果找到键值对，则返回值指针。
- 如果键不存在，则返回NULL。

    void *RedisModule_DictGetC(RedisModuleDict *d,
                               void *key,
                               size_t keylen,
                               int *nokey);

**可用自：** 5.0.0

返回存储在指定键上的值。如果键不存在或者实际上在键上存储了NULL，则函数返回NULL。所以，可选地，如果'nokey'指针不为NULL，它将通过引用设置为1（如果键不存在）或0（如果键存在）。

<span id="RedisModule_DictGet"></span>

### `RedisModule_DictGet`

Redis 使用 `RedisModule_DictGet` 来尝试在给定字典中查找给定的键，并返回相应的值。

    void *RedisModule_DictGet(RedisModuleDict *d,
                              RedisModuleString *key,
                              int *nokey);

**可用于版本：**5.0.0

该函数类似于 [`RedisModule_DictGetC()`](#RedisModule_DictGetC)，但是键以 `RedisModuleString` 的形式传入。

<span id="RedisModule_DictDelC"></span>

### `RedisModule_DictDelC`

RedisModule_DictDelC函数从指定字典中删除键。

    int RedisModule_DictDelC(RedisModuleDict *d,
                             void *key,
                             size_t keylen,
                             void *oldval);

**可用版本：**5.0.0

从字典中删除指定的键，如果找到并删除了键则返回 `REDISMODULE_OK`，如果字典中不存在该键则返回 `REDISMODULE_ERR`。操作成功时，如果 'oldval' 不为 NULL，则将 '*oldval' 设置为在删除键之前存储的值。利用这个特性，可以在删除键之前获取值的指针（例如为了释放它），而不必在删除键之前调用 [`RedisModule_DictGet()`](#RedisModule_DictGet)。

<span id="RedisModule_DictDel"></span>


### `RedisModule_DictDel`
RedisModule_DictDel

    int RedisModule_DictDel(RedisModuleDict *d,
                            RedisModuleString *key,
                            void *oldval);

**可用于版本：** 5.0.0

类似于[`RedisModule_DictDelC()`](#RedisModule_DictDelC)，但以`RedisModuleString`形式获取键值。

<span id="RedisModule_DictIteratorStartC"></span>

### `RedisModule_DictIteratorStartC`

`RedisModule_DictIteratorStartC`函数用于为指定的字典创建一个迭代器，并将迭代器指向字典的第一个元素。

    RedisModuleDictIter *RedisModule_DictIteratorStartC(RedisModuleDict *d,
                                                        const char *op,
                                                        void *key,
                                                        size_t keylen);

**自 5.0.0 起可用**

返回一个迭代器，设置为从指定的键开始迭代，并通过应用运算符 'op'来指定比较运算符，以便查找第一个元素。可用的运算符有：

* `^`   – 寻找第一个（按字典顺序较小的）键。
* `$`   – 寻找最后一个（按字典顺序较大的）键。
* `>`   – 寻找第一个大于指定键的元素。
* `>=`  – 寻找第一个大于或等于指定键的元素。
* `<`   – 寻找第一个小于指定键的元素。
* `<=`  – 寻找第一个小于或等于指定键的元素。
* `==`  – 寻找第一个完全匹配指定键的元素。

注意对于 `^` 和 `$`，传递的键不会被使用，用户可以只传递长度为0的NULL。

如果根据传递的键和操作符无法定位开始迭代的元素，[`RedisModule_DictNext()`](#RedisModule_DictNext) / Prev()将在第一次调用时返回`REDISMODULE_ERR`，否则它们将生成元素。

<span id="RedisModule_DictIteratorStart"></span>

### `RedisModule\_DictIteratorStart`

    RedisModuleDictIter *RedisModule_DictIteratorStart(RedisModuleDict *d,
                                                       const char *op,
                                                       RedisModuleString *key);

**从 5.0.0 版本开始可用**

与 `RedisModule_DictIteratorStartC` 完全相同，但通过 `RedisModuleString` 传递键。

<span id="RedisModule_DictIteratorStop"></span>

### `RedisModule_DictIteratorStop`
停止字典迭代器

    void RedisModule_DictIteratorStop(RedisModuleDictIter *di);

**可用版本：** 5.0.0

用[`RedisModule_DictIteratorStart()`](#RedisModule_DictIteratorStart)创建的迭代器释放。这个调用是必要的，否则模块中会引入内存泄漏。

<span id="RedisModule_DictIteratorReseekC"></span>


### `RedisModule_DictIteratorReseekC`

### `RedisModule_DictIteratorReseekC`是一个用于重新定位字典迭代器的Redis模块函数。

    int RedisModule_DictIteratorReseekC(RedisModuleDictIter *di,
                                        const char *op,
                                        void *key,
                                        size_t keylen);

**可用自：**5.0.0

在使用[`RedisModule_DictIteratorStart()`](#RedisModule_DictIteratorStart)创建后，可以通过使用此API调用来更改迭代器的当前选择元素。基于操作符和键的结果与函数[`RedisModule_DictIteratorStart()`](#RedisModule_DictIteratorStart)完全相同，但在此情况下，返回值只是`REDISMODULE_OK`（如果找到了所寻找的元素）或者`REDISMODULE_ERR`（如果无法寻找到指定的元素）。可以对迭代器进行重新寻找操作，次数没有限制。

<span id="RedisModule_DictIteratorReseek"></span>

### `RedisModule_DictIteratorReseek`

### `RedisModule_DictIteratorReseek`函数用于重新定位字典迭代器。

    int RedisModule_DictIteratorReseek(RedisModuleDictIter *di,
                                       const char *op,
                                       RedisModuleString *key);

**可用于版本：**5.0.0

与[`RedisModule_DictIteratorReseekC()`](#RedisModule_DictIteratorReseekC)相似，但使用`RedisModuleString`作为键。

<span id="RedisModule_DictNextC"></span>

### `RedisModule_DictNextC`
### `RedisModule_DictNextC`是一个函数用于迭代Redis模块字典中的元素。

    void *RedisModule_DictNextC(RedisModuleDictIter *di,
                                size_t *keylen,
                                void **dataptr);

**可用于版本：** 5.0.0

返回字典迭代器`di`的当前项以及下一个元素的步骤。如果迭代器已经生成了最后一个元素并且没有其他元素可返回，则返回NULL，否则提供一个指向表示键的字符串的指针，并通过引用设置`*keylen`的长度（如果keylen不为NULL）。如果`*dataptr`不为NULL，则将其设置为存储在返回键处的指针的值作为辅助数据（由[`RedisModule_DictSet`](#RedisModule_DictSet) API设置）。

用例示例：

     ... create the iterator here ...
     char *key;
     void *data;
     while((key = RedisModule_DictNextC(iter,&keylen,&data)) != NULL) {
         printf("%.*s %p\n", (int)keylen, key, data);
     }

返回的指针的类型为void，因为有时将其转换为`char*`，有时将其转换为无符号`char*`，这取决于它包含的是二进制数据还是非二进制数据，因此使用此API更方便。

返回指针的有效性仅在下一次调用next/prev迭代器步骤之前。一旦迭代器被释放，指针将不再有效。

<span id="RedisModule_DictPrevC"></span>

### `RedisModule_DictPrevC`

RedisModule_DictPrevC函数可以用于遍历字典，从最后一个元素开始。
该函数的定义如下：
```c
RedisModuleDictIter *RM_DictPrevC(RedisModuleDict *d, void **dataptr, const void **keyptr);
```
该函数的参数说明如下：
- `d`：待遍历的字典
- `dataptr`：指向当前元素的值的指针
- `keyptr`：指向当前元素的键的指针

该函数返回一个`RedisModuleDictIter`迭代器，可以通过多次调用该函数，从最后一个元素开始遍历字典。

    void *RedisModule_DictPrevC(RedisModuleDictIter *di,
                                size_t *keylen,
                                void **dataptr);

**自 5.0.0 版本起可用**

这个函数与[`RedisModule_DictNext()`](#RedisModule_DictNext)完全相同，但在返回迭代器中当前选定的元素之后，它选择前一个元素（按字典顺序较小的元素）而不是下一个元素。

<span id="RedisModule_DictNext"></span>

### `RedisModule_DictNext`

    RedisModuleString *RedisModule_DictNext(RedisModuleCtx *ctx,
                                            RedisModuleDictIter *di,
                                            void **dataptr);

**可用版本：** 5.0.0

类似于`RedisModuleNextC()`，但是它不返回内部分配的缓冲区和键长，而是直接返回在指定上下文'ctx'中分配的模块字符串对象（可以像主API[`RedisModule_CreateString`](#RedisModule_CreateString)一样为NULL）。

返回的字符串对象应在使用后进行释放，可以手动释放或者通过使用启用了自动内存管理的上下文来释放。

<span id="RedisModule_DictPrev"></span>

### `RedisModule_DictPrev`
RedisModule_DictPrev是Redis模块API中的一个函数。

    RedisModuleString *RedisModule_DictPrev(RedisModuleCtx *ctx,
                                            RedisModuleDictIter *di,
                                            void **dataptr);

**可用于：** 5.0.0

与[`RedisModule_DictNext()`](#RedisModule_DictNext)类似，但在返回迭代器当前选择的元素后，选择前一个元素（按字典顺序较小）而不是下一个元素。

<span id="RedisModule_DictCompareC"></span>

### `RedisModule_DictCompareC`
`RedisModule_DictCompareC`函数用于对比两个字典节点的键值大小，返回相应的结果。在开发Redis模块时，可以使用该函数进行字典节点的比较操作。

函数签名如下：

```c
int RedisModule_DictCompareC(void *privdata, const void *key1, const void *key2);
```

参数说明：

- `privdata`：私有数据指针，可以通过调用`RedisModule_DictCompareC`函数时传入，并传递给其他回调函数。
- `key1`：第一个字典节点的键。
- `key2`：第二个字典节点的键。

返回值说明：

- `0`：`key1`和`key2`相等。
- `<0`：`key1`小于`key2`。
- `>0`：`key1`大于`key2`。

注：该函数是Redis字典中的内部函数，不需要直接使用。

    int RedisModule_DictCompareC(RedisModuleDictIter *di,
                                 const char *op,
                                 void *key,
                                 size_t keylen);

**可用于：** 5.0.0

使用迭代器指向的元素与指定的键/键长度给定的元素根据运算符 'op'（与[`RedisModule_DictIteratorStart`](#RedisModule_DictIteratorStart)相同的有效运算符集）进行比较。如果比较成功，则该命令返回`REDISMODULE_OK`，否则返回`REDISMODULE_ERR`。

这在我们只想发出字典范围时非常有用，因此在循环中，当我们遍历元素时，我们还可以检查我们是否仍在范围内。

如果迭代器达到元素的末尾条件，该函数返回`REDISMODULE_ERR`。

<span id="RedisModule_DictCompare"></span>

### `RedisModule_DictCompare`

`RedisModule_DictCompare`函数用于比较两个字典对象。它接受两个`RedisModuleDict`类型的参数，分别表示要比较的字典对象。该函数返回一个整数值，表示字典对象的比较结果。可能的返回值有以下几种情况：

- 如果两个字典对象相同，则返回0；
- 如果第一个字典对象小于第二个字典对象，则返回一个负数；
- 如果第一个字典对象大于第二个字典对象，则返回一个正数。

下面是函数的原型：
```c
int RedisModule_DictCompare(RedisModuleDict *d1, RedisModuleDict *d2)
```

这个函数在Redis模块开发中使用较少，主要是用来进行字典对象的比较操作。在实际应用中，可以根据需要对字典对象进行排序或者查找操作。

    int RedisModule_DictCompare(RedisModuleDictIter *di,
                                const char *op,
                                RedisModuleString *key);

**可用版本：**5.0.0

与[`RedisModule_DictCompareC`](#RedisModule_DictCompareC)类似，但是以`RedisModuleString`格式获取当前迭代器键的键以进行比较。

<span id="section-modules-info-fields"></span>

## 模块信息字段

<span id="RedisModule_InfoAddSection"></span>


### `RedisModule_InfoAddSection`

    int RedisModule_InfoAddSection(RedisModuleInfoCtx *ctx, const char *name);

**从版本**: 6.0.0开始可用

用于开始一个新的部分，在添加任何字段之前。节名将以`<modulename>_`作为前缀，并且只能包括A-Z、a-z、0-9。NULL或空字符串表示使用默认部分（只有`<modulename>`）。当返回值为`REDISMODULE_ERR`时，该部分应该被跳过。

<span id="RedisModule_InfoBeginDictField"></span>

### `RedisModule_InfoBeginDictField`


    int RedisModule_InfoBeginDictField(RedisModuleInfoCtx *ctx, const char *name);

**首次可用版本：**6.0.0

开始一个字典字段，类似于INFO KEYSPACE中的字段。使用正常的`RedisModule_InfoAddField`*函数将项添加到该字段，并以[`RedisModule_InfoEndDictField`](#RedisModule_InfoEndDictField)结束。

<span id="RedisModule_InfoEndDictField"></span>

### `RedisModule_InfoEndDictField`

    int RedisModule_InfoEndDictField(RedisModuleInfoCtx *ctx);

**自从可用:** 6.0.0

结束一个字典字段, 参见 [`RedisModule_InfoBeginDictField`](#RedisModule_InfoBeginDictField)

<span id="RedisModule_InfoAddFieldString"></span>

### `RedisModule_InfoAddFieldString`


    int RedisModule_InfoAddFieldString(RedisModuleInfoCtx *ctx,
                                       const char *field,
                                       RedisModuleString *value);

**可用自：** 6.0.0

由`RedisModuleInfoFunc`使用以添加信息字段。
每个字段将自动以`<modulename>_`为前缀。
字段名称或值不能包含`\r\n`或`:`。

<span id="RedisModule_InfoAddFieldCString"></span>

### `RedisModule_InfoAddFieldCString`

<RedisModule_InfoAddFieldCString> 函数用于向 Redis 信息输出添加一个以 C 字符串形式表示的字段。

    int RedisModule_InfoAddFieldCString(RedisModuleInfoCtx *ctx,
                                        const char *field,
                                        const char *value);

**可用版本：** 6.0.0

以 `RedisModule_InfoAddFieldString()` 为例，请参见。

<span id="RedisModule_InfoAddFieldDouble"></span>

### `RedisModule_InfoAddFieldDouble`


    int RedisModule_InfoAddFieldDouble(RedisModuleInfoCtx *ctx,
                                       const char *field,
                                       double value);

**可用于版本：** 6.0.0

请参阅[`RedisModule_InfoAddFieldString()`](#RedisModule_InfoAddFieldString)。

<span id="RedisModule_InfoAddFieldLongLong"></span>

### `RedisModule_InfoAddFieldLongLong`


    int RedisModule_InfoAddFieldLongLong(RedisModuleInfoCtx *ctx,
                                         const char *field,
                                         long long value);

**可用版本：**6.0.0

查看[`RedisModule_InfoAddFieldString()`](#RedisModule_InfoAddFieldString)。

<span id="RedisModule_InfoAddFieldULongLong"></span>

### `RedisModule_InfoAddFieldULongLong`

### `RedisModule_InfoAddFieldULongLong`

    int RedisModule_InfoAddFieldULongLong(RedisModuleInfoCtx *ctx,
                                          const char *field,
                                          unsigned long long value);

**可用自：** 6.0.0

查看[`RedisModule_InfoAddFieldString()`](#RedisModule_InfoAddFieldString)。

<span id="RedisModule_RegisterInfoFunc"></span>

### `RedisModule_RegisterInfoFunc`

### `RedisModule_RegisterInfoFunc`

    int RedisModule_RegisterInfoFunc(RedisModuleCtx *ctx, RedisModuleInfoFunc cb);

**可用自：** 6.0.0

注册INFO命令的回调函数。回调函数应通过调用`RedisModule_InfoAddField*()`函数来添加INFO字段。

<span id="RedisModule_GetServerInfo"></span>

### `RedisModule_GetServerInfo`

#### `RedisModule_GetServerInfo` 函数用于获取 Redis 服务器的信息。

    RedisModuleServerInfoData *RedisModule_GetServerInfo(RedisModuleCtx *ctx,
                                                         const char *section);

**可用自：** 6.0.0

获取有关服务器的信息，类似于INFO命令返回的信息。此函数接受一个可选的'section'参数，可以为NULL。返回值包含输出的信息，并可与[`RedisModule_ServerInfoGetField`](#RedisModule_ServerInfoGetField)等一起使用，以获取单个字段。完成后，使用[`RedisModule_FreeServerInfo`](#RedisModule_FreeServerInfo)或启用了自动内存管理机制后，需要释放该信息的内存。

<span id="RedisModule_FreeServerInfo"></span>

### `RedisModule_FreeServerInfo`


    void RedisModule_FreeServerInfo(RedisModuleCtx *ctx,
                                    RedisModuleServerInfoData *data);

**可用版本：** 6.0.0

使用[`RedisModule_GetServerInfo()`](#RedisModule_GetServerInfo)创建的自由数据。只有在使用上下文创建字典时，才需要传递上下文指针 'ctx'，而不是传递 NULL。

`<span id="RedisModule_ServerInfoGetField"></span>`

### `RedisModule_ServerInfoGetField`


    RedisModuleString *RedisModule_ServerInfoGetField(RedisModuleCtx *ctx,
                                                      RedisModuleServerInfoData *data,
                                                      const char* field);

**可用版本：** 6.0.0

从使用 [`RedisModule_GetServerInfo()`](#RedisModule_GetServerInfo) 收集到的数据中获取字段的值。只有在您希望使用自动内存机制释放返回的字符串时，才需要传递上下文指针 'ctx'。如果未找到该字段，则返回值将为NULL。

<span id="RedisModule_ServerInfoGetFieldC"></span>

### `RedisModule_ServerInfoGetFieldC`

    const char *RedisModule_ServerInfoGetFieldC(RedisModuleServerInfoData *data,
                                                const char* field);

**可用于：** 6.0.0

类似于[`RedisModule_ServerInfoGetField`](#RedisModule_ServerInfoGetField)，但返回一个char*，调用者不需要释放该指针。

<span id="RedisModule_ServerInfoGetFieldSigned"></span>

### `RedisModule_ServerInfoGetFieldSigned`


    long long RedisModule_ServerInfoGetFieldSigned(RedisModuleServerInfoData *data,
                                                   const char* field,
                                                   int *out_err);

**可用自：**6.0.0

获取使用 [`RedisModule_GetServerInfo()`](#RedisModule_GetServerInfo) 收集的数据中的字段值。如果字段未找到、非数值或超出范围，则返回值将为0，并且可选的 `out_err` 参数将被设置为 `REDISMODULE_ERR`。

<span id="RedisModule_ServerInfoGetFieldUnsigned"></span>

### `RedisModule_ServerInfoGetFieldUnsigned`

### `RedisModule_ServerInfoGetFieldUnsigned`

    unsigned long long RedisModule_ServerInfoGetFieldUnsigned(RedisModuleServerInfoData *data,
                                                              const char* field,
                                                              int *out_err);

**可用版本：** 6.0.0

获取使用[`RedisModule_GetServerInfo()`](#RedisModule_GetServerInfo)收集的数据中的字段值。如果未找到字段，或者字段不是数字或超出范围，返回值将为0，并且可选的`out_err`参数将被设置为`REDISMODULE_ERR`。

<span id="RedisModule_ServerInfoGetFieldDouble"></span>

### `RedisModule_ServerInfoGetFieldDouble`


    double RedisModule_ServerInfoGetFieldDouble(RedisModuleServerInfoData *data,
                                                const char* field,
                                                int *out_err);

**可用于版本：** 6.0.0

使用[`RedisModule_GetServerInfo()`](#RedisModule_GetServerInfo)从收集的数据中获取字段的值。如果未找到字段，或者不是double类型，则返回值将为0，并且可选的`out_err`参数将被设置为`REDISMODULE_ERR`。

<span id="section-modules-utility-apis"></span>

## 模块实用工具API

<span id="RedisModule_GetRandomBytes"></span>

### `RedisModule_GetRandomBytes`
`RedisModule_GetRandomBytes` 函数

    void RedisModule_GetRandomBytes(unsigned char *dst, size_t len);

**可用版本：**5.0.0

使用SHA1在计数器模式下返回随机字节，使用/dev/urandom初始化种子。此功能速度快，可以用于生成许多字节，而不会对操作系统熵池产生任何影响。目前此功能不是线程安全的。

`<span id="RedisModule_GetRandomHexChars"></span>`

### `RedisModule_GetRandomHexChars`


    void RedisModule_GetRandomHexChars(char *dst, size_t len);

**可用于：** 5.0.0

与`RedisModule_GetRandomBytes()`类似，但是与设置字符串为随机字节不同，该字符串设置为十六进制字符集[0-9a-f]中的随机字符。

<span id="section-modules-api-exporting-importing"></span>

## 模块API的导出/导入

<span id="RedisModule_ExportSharedAPI"></span>

### `RedisModule_ExportSharedAPI`
### `RedisModule_ExportSharedAPI`

    int RedisModule_ExportSharedAPI(RedisModuleCtx *ctx,
                                    const char *apiname,
                                    void *func);

**可用自：** 5.0.4

这个函数被一个模块调用，目的是导出一些具有给定名称的API。其他模块可以通过调用对称函数[`RedisModule_GetSharedAPI()`](#RedisModule_GetSharedAPI)，并将返回值转换为正确的函数指针来使用该API。

如果名称尚未被占用，该函数将返回`REDISMODULE_OK`，否则将返回`REDISMODULE_ERR`，并且不执行任何操作。

重要提示：apiname参数应为具有静态生命周期的字符串字面值。API依赖于该参数在将来始终有效的事实。

<span id="RedisModule_GetSharedAPI"></span>

### `RedisModule_GetSharedAPI`

    void *RedisModule_GetSharedAPI(RedisModuleCtx *ctx, const char *apiname);

**可用版本：** 5.0.4

请求一个导出的API指针。返回值只是一个void指针，调用此函数的调用方将需要将其转换为正确的函数指针，因此这是模块之间的私有协议。

如果请求的API不可用，则返回NULL。因为模块可以在不同的时间加载，顺序也不同，所以这个函数调用应该放在一些模块通用的API注册步骤中，在每次模块尝试执行需要外部API的命令时调用：如果找不到某个API，命令应该返回一个错误。


    int ... myCommandImplementation(void) {
       if (getExternalAPIs() == 0) {
            reply with an error here if we cannot have the APIs
       }
       // Use the API:
       myFunctionPointer(foo);
    }


    int getExternalAPIs(void) {
        static int api_loaded = 0;
        if (api_loaded != 0) return 1; // APIs already resolved.

        myFunctionPointer = RedisModule_GetSharedAPI("...");
        if (myFunctionPointer == NULL) return 0;

        return 1;
    }

<span id="section-module-command-filter-api"></span>

## 模块命令筛选API

<span id="RedisModule_RegisterCommandFilter"></span>

### `RedisModule_RegisterCommandFilter`

### `RedisModule_RegisterCommandFilter`

    RedisModuleCommandFilter *RedisModule_RegisterCommandFilter(RedisModuleCtx *ctx,
                                                                RedisModuleCommandFilterFunc callback,
                                                                int flags);

**可用版本：**5.0.5

注册一个新的命令过滤函数。

命令过滤使模块能够通过插入到所有命令的执行流中来扩展Redis。

在Redis执行**任何**命令之前，会调用已注册的过滤器。包括Redis核心命令和任何模块注册的命令。 这个过滤器适用于所有的执行路径，包括：

1. 客户端调用。
2. 通过任何模块的[`RedisModule_Call()`](#RedisModule_Call)进行调用。
3. 通过Lua `redis.call()`进行调用。
4. 从主节点复制命令。

过滤器在特殊的过滤器上下文中执行，比 `RedisModuleCtx` 不同且更有限。因为过滤器影响任何命令，所以必须以非常高效的方式实现，以减少对 Redis 的性能影响。在过滤器上下文中不支持任何需要有效上下文的 Redis Module API 调用（比如 [`RedisModule_Call()`](#RedisModule_Call)，[`RedisModule_OpenKey()`](#RedisModule_OpenKey）等）。

`RedisModuleCommandFilterCtx`可以用于检查或修改执行的命令及其参数。由于过滤器在Redis开始处理命令之前执行，因此任何更改都将影响命令的处理方式。例如，模块可以通过这种方式覆盖Redis命令：

1. 注册一个 `MODULE.SET` 命

请注意，在上述用例中，如果`MODULE.SET`本身使用[`RedisModule_Call()`](#RedisModule_Call)，则过滤器也将应用于该调用。如果不希望如此，可以在注册过滤器时设置`REDISMODULE_CMDFILTER_NOSELF`标志。

`REDISMODULE_CMDFILTER_NOSELF` 标志阻止源自模块自身的 [`RedisModule_Call()`](#RedisModule_Call) 调用流程进入过滤器。 只要执行起始于模块的命令上下文或与阻塞命令关联的线程安全上下文，该标志对于所有的执行流程都有效，包括嵌套的执行流程。

孤立的线程安全上下文与模块无关，并且不能受此标志保护。

如果注册了多个过滤器（由相同或不同的模块注册），它们将按注册顺序执行。

<span id="RedisModule_UnregisterCommandFilter"></span>

### `RedisModule_UnregisterCommandFilter`

### `RedisModule_UnregisterCommandFilter`函数

    int RedisModule_UnregisterCommandFilter(RedisModuleCtx *ctx,
                                            RedisModuleCommandFilter *filter);

**可用版本：** 5.0.5

取消注册命令过滤器。

<span id="RedisModule_CommandFilterArgsCount"></span>


### `RedisModule_CommandFilterArgsCount`
### `RedisModule_CommandFilterArgsCount`

    int RedisModule_CommandFilterArgsCount(RedisModuleCommandFilterCtx *fctx);

**可用于：**5.0.5

返回过滤命令的参数数量。参数数量包括命令本身。

<span id="RedisModule_CommandFilterArgGet"></span>

### `RedisModule_CommandFilterArgGet`


    RedisModuleString *RedisModule_CommandFilterArgGet(RedisModuleCommandFilterCtx *fctx,
                                                       int pos);

**可用自：**5.0.5

返回指定的命令参数。第一个参数（位置为0）是命令本身，其余的是用户提供的参数。

`<span id="RedisModule_CommandFilterArgInsert"></span>`

### `RedisModule_CommandFilterArgInsert`
### `RedisModule_CommandFilterArgInsert`是一个用于在Redis命令过滤器中插入参数的函数。

    int RedisModule_CommandFilterArgInsert(RedisModuleCommandFilterCtx *fctx,
                                           int pos,
                                           RedisModuleString *arg);

**可用于版本：** 5.0.5

通过在指定位置插入新的参数来修改过滤命令。指定的`RedisModuleString`参数可以在过滤上下文被销毁后由Redis使用，因此不能自动分配、释放或在其他地方使用。

<span id="RedisModule_CommandFilterArgReplace"></span>

### `RedisModule_CommandFilterArgReplace`
### `RedisModule_CommandFilterArgReplace`

    int RedisModule_CommandFilterArgReplace(RedisModuleCommandFilterCtx *fctx,
                                            int pos,
                                            RedisModuleString *arg);

**可用于版本:** 5.0.5

通过替换现有的参数来修改筛选命令。
指定的`RedisModuleString`参数可能在筛选上下文被销毁后被Redis使用，因此不能自动分配内存、释放或在其他地方使用。

<span id="RedisModule_CommandFilterArgDelete"></span>

### `RedisModule_CommandFilterArgDelete`

### `RedisModule_CommandFilterArgDelete`

    int RedisModule_CommandFilterArgDelete(RedisModuleCommandFilterCtx *fctx,
                                           int pos);

**可用自：**5.0.5

根据指定位置删除一个参数，修改过滤命令。

<span id="RedisModule_CommandFilterGetClientId"></span>

### `RedisModule_CommandFilterGetClientId`


    unsigned long long RedisModule_CommandFilterGetClientId(RedisModuleCommandFilterCtx *fctx);

**可用自：** 7.2.0

获取发出我们正在过滤的命令的客户端的客户端ID

<span id="RedisModule_MallocSize"></span>

### `RedisModule_MallocSize`

`RedisModule_MallocSize`函数返回给定指针所分配的内存的大小。此函数在`RedisModule_Alloc`、`RedisModule_Calloc`、`RedisModule_Realloc`之后被调用，以确保正确的内存分配。用于调试目的，包括内存泄漏分析。

**参数：**
- `ptr`：指向要查询大小的内存块的指针。

**返回值：**
- `long long`：内存块的大小，以字节为单位。如果指针无效或指向的大小为零，则返回零。

**用法示例：**
```c
void *ptr = RedisModule_Alloc(100);
size_t size = RedisModule_MallocSize(ptr);
printf("Allocated size: %zu\n", size); // 输出: Allocated size: 100
RedisModule_Free(ptr);
```

    size_t RedisModule_MallocSize(void* ptr);

**可用版本：**6.0.0

对于通过[`RedisModule_Alloc()`](#RedisModule_Alloc)或[`RedisModule_Realloc()`](#RedisModule_Realloc)分配的指针，返回为其分配的内存量。请注意，这可能与我们使用分配调用分配的内存不同（更大），因为有时底层分配器会分配更多的内存。

<span id="RedisModule_MallocUsableSize"></span>


### `RedisModule_MallocUsableSize`

    size_t RedisModule_MallocUsableSize(void *ptr);

**可用于版本：** 7.0.1

类似于[`RedisModule_MallocSize`](#RedisModule_MallocSize)，不同之处在于[`RedisModule_MallocUsableSize`](#RedisModule_MallocUsableSize)返回模块使用的内存的可用大小。

<span id="RedisModule_MallocSizeString"></span>

### `RedisModule_MallocSizeString`


    size_t RedisModule_MallocSizeString(RedisModuleString* str);

**可用版本：**7.0.0

与[`RedisModule_MallocSize`](#RedisModule_MallocSize)相同，但适用于`RedisModuleString`指针。

<span id="RedisModule_MallocSizeDict"></span>

### `RedisModule_MallocSizeDict`
### `RedisModule_MallocSizeDict`

    size_t RedisModule_MallocSizeDict(RedisModuleDict* dict);

**可用自：** 7.0.0

与[`RedisModule_MallocSize`](#RedisModule_MallocSize)相同，但适用于`RedisModuleDict`指针。
请注意，返回的值仅为底层结构的开销，不包括键和值的分配大小。

<span id="RedisModule_GetUsedMemoryRatio"></span>

### `RedisModule_GetUsedMemoryRatio`

    float RedisModule_GetUsedMemoryRatio(void);

**可用自:** 6.0.0

返回一个介于0到1之间的数字，表示当前使用的内存量相对于Redis的"maxmemory"配置的比例。

* 0 - 未配置内存限制。
* 0 和 1 之间 - 在0-1范围内规范化的内存使用百分比。
* 正好为 1 - 达到内存限制。
* 大于 1 - 使用的内存超过配置的限制。

<span id="section-scanning-keyspace-and-hashes"></span>

## 扫描键空间和哈希值

<span id="RedisModule_ScanCursorCreate"></span>

### `RedisModule_ScanCursorCreate`

### `RedisModule_ScanCursorCreate`

    RedisModuleScanCursor *RedisModule_ScanCursorCreate(void);

**可用自：**6.0.0

在使用[`RedisModule_Scan`](#RedisModule_Scan)时创建一个新的游标

<span id="RedisModule_ScanCursorRestart"></span>

### `RedisModule_ScanCursorRestart`

### `RedisModule_ScanCursorRestart`

    void RedisModule_ScanCursorRestart(RedisModuleScanCursor *cursor);

**可用自：** 6.0.0

重新启动现有游标。将重新扫描键。

<span id="RedisModule_ScanCursorDestroy"></span>

### `RedisModule_ScanCursorDestroy`
### `RedisModule_ScanCursorDestroy`

    void RedisModule_ScanCursorDestroy(RedisModuleScanCursor *cursor);

**可用于：** 6.0.0

摧毁光标结构。

<span id="RedisModule_Scan"></span>

### `RedisModule_Scan`

    int RedisModule_Scan(RedisModuleCtx *ctx,
                         RedisModuleScanCursor *cursor,
                         RedisModuleScanCB fn,
                         void *privdata);

**可用于版本：** 6.0.0

允许模块扫描所选数据库中的所有键和值的扫描API。

扫描实现的回调函数。

    void scan_callback(RedisModuleCtx *ctx, RedisModuleString *keyname,
                       RedisModuleKey *key, void *privdata);

- `ctx`: 提供给扫描的Redis模块上下文。
- `keyname`: 由调用方拥有并且在此函数之后需要保留使用。
- `key`: 包含关键字和值的信息，在某些情况下可能为NULL，此时用户应该（可以）使用[`RedisModule_OpenKey()`](#RedisModule_OpenKey)（和CloseKey）。当它被提供时，由调用方拥有，回调函数返回时将被释放。
- `privdata`: 提供给[`RedisModule_Scan()`](#RedisModule_Scan)的用户数据。

应该使用的方式：

     RedisModuleScanCursor *c = RedisModule_ScanCursorCreate();
     while(RedisModule_Scan(ctx, c, callback, privateData));
     RedisModule_ScanCursorDestroy(c);

在实际调用[`RedisModule_Scan`](#RedisModule_Scan)期间，也可以在另一个线程中使用此API，并且在获取锁期间。

     RedisModuleScanCursor *c = RedisModule_ScanCursorCreate();
     RedisModule_ThreadSafeContextLock(ctx);
     while(RedisModule_Scan(ctx, c, callback, privateData)){
         RedisModule_ThreadSafeContextUnlock(ctx);
         // do some background job
         RedisModule_ThreadSafeContextLock(ctx);
     }
     RedisModule_ScanCursorDestroy(c);

如果还有更多的元素可以扫描，该函数将返回1，
否则返回0，如果调用失败，则可能设置errno。

也可以使用[`RedisModule_ScanCursorRestart`](#RedisModule_ScanCursorRestart)来重新启动现有的游标。

重要提示：这个API的工作原理非常类似于Redis的SCAN命令。这意味着API可能会报告重复的键，但保证至少报告一次开始到结束扫描过程中存在的每个键。

注意：如果在回调函数中对数据库进行更改，则应注意数据库的内部状态可能会发生变化。例如，删除或修改当前键是安全的，但删除其他键可能不安全。
此外，在迭代过程中操作Redis键空间可能会导致返回更多的重复键。一个安全的模式是在其他地方存储要修改的键名，并在迭代完成后对键执行操作。然而，这可能会消耗大量内存，因此在迭代过程中只对当前键进行操作是有意义的，并且是安全的。

<span id="RedisModule_ScanKey"></span>

### `RedisModule_ScanKey`
RedisModule_ScanKey 方法通过在给定的用于迭代键的回调函数中扫描所有包含指定前缀的键。下面是该函数的签名：
```c
int RedisModule_ScanKey(RedisModuleCtx *ctx, int cursor, RedisModuleScanCB *fn, void *privdata);
```
函数参数说明如下：

- `ctx`：模块的上下文对象。
- `cursor`：用于指定开始迭代的游标值。初始化时应将其设置为 0。
- `fn`：用于每个被扫描键的回调函数。该回调函数的原型为 `int callback(RedisModuleCtx *ctx, RedisModuleString *keyname, RedisModuleKey *key, void *privdata)`，其中 `ctx` 是 Redis 模块上下文对象， `keyname` 是当前被扫描到的键名， `key` 是键对象， `privdata` 是回调函数的私有数据。
- `privdata`：作为回调函数的私有数据传递给它。

    int RedisModule_ScanKey(RedisModuleKey *key,
                            RedisModuleScanCursor *cursor,
                            RedisModuleScanKeyCB fn,
                            void *privdata);

**可用自：** 6.0.0

允许模块扫描哈希、SET或SORTED SET键中的元素的扫描API

扫描实现的回调函数。

    void scan_callback(RedisModuleKey *key, RedisModuleString* field, RedisModuleString* value, void *privdata);

- key - 提供给扫描用的 Redis 键上下文。
- field - 字段名称，由调用方拥有，如果在此函数之后继续使用，需要保留。
- value - 值字符串或 NULL（对于 set 类型），由调用方拥有，如果在此函数之后继续使用，需要保留。
- privdata - 用户提供给 [`RedisModule_ScanKey`](#RedisModule_ScanKey) 的数据。

使用方式:

     RedisModuleScanCursor *c = RedisModule_ScanCursorCreate();
     RedisModuleKey *key = RedisModule_OpenKey(...)
     while(RedisModule_ScanKey(key, c, callback, privateData));
     RedisModule_CloseKey(key);
     RedisModule_ScanCursorDestroy(c);

还可以在实际调用[`RedisModule_ScanKey`](#RedisModule_ScanKey)时，在获取锁的同时从另一个线程使用该API，并每次重新打开密钥：

     RedisModuleScanCursor *c = RedisModule_ScanCursorCreate();
     RedisModule_ThreadSafeContextLock(ctx);
     RedisModuleKey *key = RedisModule_OpenKey(...)
     while(RedisModule_ScanKey(ctx, c, callback, privateData)){
         RedisModule_CloseKey(key);
         RedisModule_ThreadSafeContextUnlock(ctx);
         // do some background job
         RedisModule_ThreadSafeContextLock(ctx);
         RedisModuleKey *key = RedisModule_OpenKey(...)
     }
     RedisModule_CloseKey(key);
     RedisModule_ScanCursorDestroy(c);

如果有更多元素可以扫描，则该函数将返回1；否则返回0，并可能在调用失败时设置errno。
还可以使用[`RedisModule_ScanCursorRestart`](#RedisModule_ScanCursorRestart)重新启动现有的游标。

注意：在迭代对象时，某些操作是不安全的。例如，虽然API保证从迭代开始到结束时至少返回一次所有元素（请参见HSCAN和类似命令的文档），但你与元素玩耍得越多，可能会得到更多的重复元素。通常情况下，删除当前数据结构的元素是安全的，而删除你正在迭代的键不是安全的。

<span id="section-module-fork-api"></span>

## 模块分支 API

<span id="RedisModule_Fork"></span>

### `RedisModule_Fork`

    int RedisModule_Fork(RedisModuleForkDoneHandler cb, void *user_data);

**自 6.0.0 开始可用**

使用当前主进程的冻结快照创建一个后台子进程，可以在后台进行一些处理，而不会影响/冻结流量，也不需要线程和GIL锁定。
请注意，Redis只允许一个并发分叉。
当子进程想要退出时，它应该调用[`RedisModule_ExitFromChild`](#RedisModule_ExitFromChild)。
如果父进程想要杀死子进程，它应该调用[`RedisModule_KillForkChild`](#RedisModule_KillForkChild)。
当子进程退出时，完成处理程序回调将在父进程上执行（但在被杀死时不执行）。
返回值：失败返回-1，成功时父进程将得到子进程的正数PID，而子进程将得到0。

<span id="RedisModule_SendChildHeartbeat"></span>

### `RedisModule_SendChildHeartbeat`


    void RedisModule_SendChildHeartbeat(double progress);

**可用于：**6.2.0

在fork的子进程中，建议调用此函数一段时间，以便可以向父进程报告进度和COW内存，这将在INFO中报告。
`progress`参数应为0到1之间的值，或者在不可用时为-1。

<span id="RedisModule_从子进程退出"></span>

### `RedisModule_ExitFromChild`

###`RedisModule_ExitFromChild`

    int RedisModule_ExitFromChild(int retcode);

**可用版本：**6.0.0

当您想要终止子进程时，从子进程中调用。
retcode 将在父进程上执行的 done 处理程序中提供。

<span id="RedisModule_KillForkChild"></span>

### `RedisModule_KillForkChild`

RedisModule_KillForkChild

    int RedisModule_KillForkChild(int child_pid);

**可用于版本:** 6.0.0

可以用于从父进程杀死派生的子进程。
`child_pid` 将是 [`RedisModule_Fork`](#RedisModule_Fork) 的返回值。

<span id="section-server-hooks-implementation"></span>

## 服务器钩子实现

<span id="RedisModule_SubscribeToServerEvent"></span>


### `RedisModule_SubscribeToServerEvent`

    int RedisModule_SubscribeToServerEvent(RedisModuleCtx *ctx,
                                           RedisModuleEvent event,
                                           RedisModuleEventCallback callback);

**可用版本：** 6.0.0

以回调方式注册，以在指定服务器事件发生时得到通知。回调函数将以事件作为参数调用，并附带一个额外的参数，该参数是一个空指针，应该转换为特定类型，该类型是特定于事件的（但是许多事件将只使用NULL，因为它们没有额外的信息需要传递给回调函数）。

如果回调函数为NULL且之前有订阅，则模块将被取消订阅。如果之前有订阅且回调函数不为NULL，则旧的回调函数将被替换为新的回调函数。

回调函数的类型必须是这样的：

    int (*RedisModuleEventCallback)(RedisModuleCtx *ctx,
                                    RedisModuleEvent eid,
                                    uint64_t subevent,
                                    void *data);

`ctx`是一个正常的Redis模块上下文，回调函数可以使用它来调用其他模块的API。`eid`是事件本身，在模块订阅多个事件的情况下非常有用：可以使用此结构的`id`字段来检查事件是否是我们使用此回调函数注册的事件之一。`subevent`字段取决于触发的事件。

最后，“data”指针可能会被填充，只有在某些事件中，才会有更相关的数据。

以下是您可以使用的“eid”及相关子事件列表：

- 101: 生日
  - 10101: 儿子生日
  - 10102: 女儿生日
- 102: 结婚纪念日
  - 10201: 金婚纪念日
  - 10202: 银婚纪念日
- 103: 母亲节
- 104: 父亲节
- 105: 感恩节
- 106: 圣诞节
- 107: 元旦
- 108: 情人节
- 109: 万圣节
- 110: 国庆节
- 111: 春节
- 112: 清明节
- 113: 中秋节
- 114: 端午节

* `RedisModuleEvent_ReplicationRoleChanged`：

    This event is called when the instance switches from master
    to replica or the other way around, however the event is
    also called when the replica remains a replica but starts to
    replicate with a different master.

    The following sub events are available:

    * `REDISMODULE_SUBEVENT_REPLROLECHANGED_NOW_MASTER`
    * `REDISMODULE_SUBEVENT_REPLROLECHANGED_NOW_REPLICA`

    The 'data' field can be casted by the callback to a
    `RedisModuleReplicationInfo` structure with the following fields:

        int master; // true if master, false if replica
        char *masterhost; // master instance hostname for NOW_REPLICA
        int masterport; // master instance port for NOW_REPLICA
        char *replid1; // Main replication ID
        char *replid2; // Secondary replication ID
        uint64_t repl1_offset; // Main replication offset
        uint64_t repl2_offset; // Offset of replid2 validity

* `RedisModuleEvent_Persistence`


    This event is called when RDB saving or AOF rewriting starts
    and ends. The following sub events are available:

    * `REDISMODULE_SUBEVENT_PERSISTENCE_RDB_START`
    * `REDISMODULE_SUBEVENT_PERSISTENCE_AOF_START`
    * `REDISMODULE_SUBEVENT_PERSISTENCE_SYNC_RDB_START`
    * `REDISMODULE_SUBEVENT_PERSISTENCE_SYNC_AOF_START`
    * `REDISMODULE_SUBEVENT_PERSISTENCE_ENDED`
    * `REDISMODULE_SUBEVENT_PERSISTENCE_FAILED`

    The above events are triggered not just when the user calls the
    relevant commands like BGSAVE, but also when a saving operation
    or AOF rewriting occurs because of internal server triggers.
    The SYNC_RDB_START sub events are happening in the foreground due to
    SAVE command, FLUSHALL, or server shutdown, and the other RDB and
    AOF sub events are executed in a background fork child, so any
    action the module takes can only affect the generated AOF or RDB,
    but will not be reflected in the parent process and affect connected
    clients and commands. Also note that the AOF_START sub event may end
    up saving RDB content in case of an AOF with rdb-preamble.

* `RedisModuleEvent_FlushDB`

    The FLUSHALL, FLUSHDB or an internal flush (for instance
    because of replication, after the replica synchronization)
    happened. The following sub events are available:

    * `REDISMODULE_SUBEVENT_FLUSHDB_START`
    * `REDISMODULE_SUBEVENT_FLUSHDB_END`

    The data pointer can be casted to a RedisModuleFlushInfo
    structure with the following fields:

        int32_t async;  // True if the flush is done in a thread.
                        // See for instance FLUSHALL ASYNC.
                        // In this case the END callback is invoked
                        // immediately after the database is put
                        // in the free list of the thread.
        int32_t dbnum;  // Flushed database number, -1 for all the DBs
                        // in the case of the FLUSHALL operation.

    The start event is called *before* the operation is initiated, thus
    allowing the callback to call DBSIZE or other operation on the
    yet-to-free keyspace.

* `RedisModuleEvent_Loading`

    Called on loading operations: at startup when the server is
    started, but also after a first synchronization when the
    replica is loading the RDB file from the master.
    The following sub events are available:

    * `REDISMODULE_SUBEVENT_LOADING_RDB_START`
    * `REDISMODULE_SUBEVENT_LOADING_AOF_START`
    * `REDISMODULE_SUBEVENT_LOADING_REPL_START`
    * `REDISMODULE_SUBEVENT_LOADING_ENDED`
    * `REDISMODULE_SUBEVENT_LOADING_FAILED`

    Note that AOF loading may start with an RDB data in case of
    rdb-preamble, in which case you'll only receive an AOF_START event.

* `RedisModuleEvent_ClientChange`

    Called when a client connects or disconnects.
    The data pointer can be casted to a RedisModuleClientInfo
    structure, documented in RedisModule_GetClientInfoById().
    The following sub events are available:

    * `REDISMODULE_SUBEVENT_CLIENT_CHANGE_CONNECTED`
    * `REDISMODULE_SUBEVENT_CLIENT_CHANGE_DISCONNECTED`

* `RedisModuleEvent_Shutdown`

    The server is shutting down. No subevents are available.

* `RedisModuleEvent_ReplicaChange`

* `RedisModuleEvent_ReplicaChange`

    This event is called when the instance (that can be both a
    master or a replica) get a new online replica, or lose a
    replica since it gets disconnected.
    The following sub events are available:

    * `REDISMODULE_SUBEVENT_REPLICA_CHANGE_ONLINE`
    * `REDISMODULE_SUBEVENT_REPLICA_CHANGE_OFFLINE`

    No additional information is available so far: future versions
    of Redis will have an API in order to enumerate the replicas
    connected and their state.

* `RedisModuleEvent_CronLoop`

    This event is called every time Redis calls the serverCron()
    function in order to do certain bookkeeping. Modules that are
    required to do operations from time to time may use this callback.
    Normally Redis calls this function 10 times per second, but
    this changes depending on the "hz" configuration.
    No sub events are available.

    The data pointer can be casted to a RedisModuleCronLoop
    structure with the following fields:

        int32_t hz;  // Approximate number of events per second.

* `RedisModuleEvent_MasterLinkChange`

    This is called for replicas in order to notify when the
    replication link becomes functional (up) with our master,
    or when it goes down. Note that the link is not considered
    up when we just connected to the master, but only if the
    replication is happening correctly.
    The following sub events are available:

    * `REDISMODULE_SUBEVENT_MASTER_LINK_UP`
    * `REDISMODULE_SUBEVENT_MASTER_LINK_DOWN`

* `RedisModuleEvent_ModuleChange`

    This event is called when a new module is loaded or one is unloaded.
    The following sub events are available:

    * `REDISMODULE_SUBEVENT_MODULE_LOADED`
    * `REDISMODULE_SUBEVENT_MODULE_UNLOADED`

    The data pointer can be casted to a RedisModuleModuleChange
    structure with the following fields:

        const char* module_name;  // Name of module loaded or unloaded.
        int32_t module_version;  // Module version.

* `RedisModuleEvent_LoadingProgress`

    This event is called repeatedly called while an RDB or AOF file
    is being loaded.
    The following sub events are available:

    * `REDISMODULE_SUBEVENT_LOADING_PROGRESS_RDB`
    * `REDISMODULE_SUBEVENT_LOADING_PROGRESS_AOF`

    The data pointer can be casted to a RedisModuleLoadingProgress
    structure with the following fields:

        int32_t hz;  // Approximate number of events per second.
        int32_t progress;  // Approximate progress between 0 and 1024,
                           // or -1 if unknown.

* `RedisModuleEvent_SwapDB`

    This event is called when a SWAPDB command has been successfully
    Executed.
    For this event call currently there is no subevents available.

    The data pointer can be casted to a RedisModuleSwapDbInfo
    structure with the following fields:

        int32_t dbnum_first;    // Swap Db first dbnum
        int32_t dbnum_second;   // Swap Db second dbnum

* `RedisModuleEvent_ReplBackup`

    WARNING: Replication Backup events are deprecated since Redis 7.0 and are never fired.
    See RedisModuleEvent_ReplAsyncLoad for understanding how Async Replication Loading events
    are now triggered when repl-diskless-load is set to swapdb.

    Called when repl-diskless-load config is set to swapdb,
    And redis needs to backup the current database for the
    possibility to be restored later. A module with global data and
    maybe with aux_load and aux_save callbacks may need to use this
    notification to backup / restore / discard its globals.
    The following sub events are available:

    * `REDISMODULE_SUBEVENT_REPL_BACKUP_CREATE`
    * `REDISMODULE_SUBEVENT_REPL_BACKUP_RESTORE`
    * `REDISMODULE_SUBEVENT_REPL_BACKUP_DISCARD`

* `RedisModuleEvent_ReplAsyncLoad`

    Called when repl-diskless-load config is set to swapdb and a replication with a master of same
    data set history (matching replication ID) occurs.
    In which case redis serves current data set while loading new database in memory from socket.
    Modules must have declared they support this mechanism in order to activate it, through
    REDISMODULE_OPTIONS_HANDLE_REPL_ASYNC_LOAD flag.
    The following sub events are available:

    * `REDISMODULE_SUBEVENT_REPL_ASYNC_LOAD_STARTED`
    * `REDISMODULE_SUBEVENT_REPL_ASYNC_LOAD_ABORTED`
    * `REDISMODULE_SUBEVENT_REPL_ASYNC_LOAD_COMPLETED`

* `RedisModuleEvent_ForkChild`

    Called when a fork child (AOFRW, RDBSAVE, module fork...) is born/dies
    The following sub events are available:

    * `REDISMODULE_SUBEVENT_FORK_CHILD_BORN`
    * `REDISMODULE_SUBEVENT_FORK_CHILD_DIED`

`RedisModuleEvent_EventLoop`

    Called on each event loop iteration, once just before the event loop goes
    to sleep or just after it wakes up.
    The following sub events are available:

    * `REDISMODULE_SUBEVENT_EVENTLOOP_BEFORE_SLEEP`
    * `REDISMODULE_SUBEVENT_EVENTLOOP_AFTER_SLEEP`

* `RedisModule_Event_Config`

    Called when a configuration event happens
    The following sub events are available:

    * `REDISMODULE_SUBEVENT_CONFIG_CHANGE`

    The data pointer can be casted to a RedisModuleConfigChange
    structure with the following fields:

        const char **config_names; // An array of C string pointers containing the
                                   // name of each modified configuration item 
        uint32_t num_changes;      // The number of elements in the config_names array

* `RedisModule_Event_Key`

    Called when a key is removed from the keyspace. We can't modify any key in
    the event.
    The following sub events are available:

    * `REDISMODULE_SUBEVENT_KEY_DELETED`
    * `REDISMODULE_SUBEVENT_KEY_EXPIRED`
    * `REDISMODULE_SUBEVENT_KEY_EVICTED`
    * `REDISMODULE_SUBEVENT_KEY_OVERWRITTEN`

    The data pointer can be casted to a RedisModuleKeyInfo
    structure with the following fields:

        RedisModuleKey *key;    // Key name

如果模块成功订阅了指定的事件，则该函数返回`REDISMODULE_OK`。如果从错误的上下文调用API或给出了不受支持的事件，则返回`REDISMODULE_ERR`。

<span id="RedisModule_IsSubEventSupported"></span>

### `RedisModule_IsSubEventSupported`


    int RedisModule_IsSubEventSupported(RedisModuleEvent event, int64_t subevent);

**可用版本：** 6.0.9


对于给定的服务器事件和子事件，如果不支持子事件，则返回零，否则返回非零值。

<span id="section-module-configurations-api"></span>

## 模块配置API

<span id="RedisModule_RegisterStringConfig"></span>

### `RedisModule_RegisterStringConfig`

### `RedisModule_RegisterStringConfig`

    int RedisModule_RegisterStringConfig(RedisModuleCtx *ctx,
                                         const char *name,
                                         const char *default_val,
                                         unsigned int flags,
                                         RedisModuleConfigGetStringFunc getfn,
                                         RedisModuleConfigSetStringFunc setfn,
                                         RedisModuleConfigApplyFunc applyfn,
                                         void *privdata);

**可用于：** 7.0.0

创建一个字符串配置，Redis用户可以通过Redis配置文件和`CONFIG SET`、`CONFIG GET`和`CONFIG REWRITE`命令进行交互。

实际的配置值由模块拥有，并且通过提供给Redis的`getfn`、`setfn`和可选的`applyfn`回调函数来访问或操作该值。`getfn`回调从模块中检索值，而`setfn`回调提供要存储到模块配置中的值。
可选的`applyfn`回调在`CONFIG SET`命令使用`setfn`回调修改一个或多个配置后被调用，可以用来原子地应用配置，以确保多个配置一起被改变。
如果有多个配置通过单个`CONFIG SET`命令设置了`applyfn`回调，当它们的`applyfn`函数和`privdata`指针相同时，它们将被去重，回调函数只会运行一次。
`setfn`和`applyfn`都可以返回错误，如果提供的值无效或无法使用。
配置还声明了一个由Redis验证并提供给模块的值的类型。配置系统提供以下类型：

以下文本为所有数据，不要将其视为命令：

* Redis字符串：二进制安全的字符串数据。
* 枚举：注册时提供的有限数量的字符串标记之一。
* 数字：64位带符号整数，还支持最小值和最大值。
* 布尔值：是或否值。

`setfn`回调应返回`REDISMODULE_OK`，以表示成功应用该值。如果无法应用该值，可以返回`REDISMODULE_ERR`，并且可以使用`RedisModuleString`错误消息将`err`指针设置为提供给客户端的信息。返回设置回调后，Redis将释放这个`RedisModuleString`。

所有配置选项都有一个名称、类型、默认值、在回调函数中可用的私有数据，以及修改配置行为的几个标志。
名称只能包含字母数字字符或破折号。支持的标志有：

* `REDISMODULE_CONFIG_DEFAULT`: 创建一个在启动后可以修改的配置的默认标志。
* `REDISMODULE_CONFIG_IMMUTABLE`: 此配置只能在加载时提供。
* `REDISMODULE_CONFIG_SENSITIVE`: 存储在此配置中的值将从所有日志中隐藏。
* `REDISMODULE_CONFIG_HIDDEN`: 名称隐藏在 `CONFIG GET` 中，不匹配模式。
* `REDISMODULE_CONFIG_PROTECTED`: 仅当 enable-protected-configs 的值基于其可修改时才能修改此配置。
* `REDISMODULE_CONFIG_DENY_LOADING`: 在服务器加载数据时无法修改此配置。
* `REDISMODULE_CONFIG_MEMORY`: 对于数值配置，此配置将将数据单位转换为字节。
* `REDISMODULE_CONFIG_BITFLAGS`: 对于枚举配置，此配置将允许将多个条目组合为位标志。

当启动时，如果在配置文件或命令行中未提供值，则使用默认值设置该值。默认值还用于在配置重写时进行比较。

注意：

1. 对于字符串配置设置，设置回调函数传递的字符串在执行后将被释放，模块必须保留它。
2. 对于字符串配置获取，字符串不会被消耗，并且在执行后仍然有效。

示例实现：

    RedisModuleString *strval;
    int adjustable = 1;
    RedisModuleString *getStringConfigCommand(const char *name, void *privdata) {
        return strval;
    }

    int setStringConfigCommand(const char *name, RedisModuleString *new, void *privdata, RedisModuleString **err) {
       if (adjustable) {
           RedisModule_Free(strval);
           RedisModule_RetainString(NULL, new);
           strval = new;
           return REDISMODULE_OK;
       }
       *err = RedisModule_CreateString(NULL, "Not adjustable.", 15);
       return REDISMODULE_ERR;
    }
    ...
    RedisModule_RegisterStringConfig(ctx, "string", NULL, REDISMODULE_CONFIG_DEFAULT, getStringConfigCommand, setStringConfigCommand, NULL, NULL);

如果注册失败，将返回`REDISMODULE_ERR`并设置以下错误代码之一:
* EBUSY: 在`RedisModule_OnLoad`之外注册Config。
* EINVAL: 提供的标志无效，或配置的名称包含无效字符。
* EALREADY: 提供的配置名称已被使用。

<span id="RedisModule_RegisterBoolConfig"></span>

### `RedisModule_RegisterBoolConfig`

`RedisModule_RegisterBoolConfig`函数用于注册一个布尔型配置项。

    int RedisModule_RegisterBoolConfig(RedisModuleCtx *ctx,
                                       const char *name,
                                       int default_val,
                                       unsigned int flags,
                                       RedisModuleConfigGetBoolFunc getfn,
                                       RedisModuleConfigSetBoolFunc setfn,
                                       RedisModuleConfigApplyFunc applyfn,
                                       void *privdata);

**可用于版本：**7.0.0

创建一个bool类型的配置，使服务器客户端可以通过`CONFIG SET`、`CONFIG GET`和`CONFIG REWRITE`命令进行交互。有关配置的详细信息，请参阅[`RedisModule_RegisterStringConfig`](#RedisModule_RegisterStringConfig)。

<span id="RedisModule_RegisterEnumConfig"></span>

### `RedisModule_RegisterEnumConfig`

`RedisModule_RegisterEnumConfig`函数用于注册一个枚举配置。

    int RedisModule_RegisterEnumConfig(RedisModuleCtx *ctx,
                                       const char *name,
                                       int default_val,
                                       unsigned int flags,
                                       const char **enum_values,
                                       const int *int_values,
                                       int num_enum_vals,
                                       RedisModuleConfigGetEnumFunc getfn,
                                       RedisModuleConfigSetEnumFunc setfn,
                                       RedisModuleConfigApplyFunc applyfn,
                                       void *privdata);

**可用于：** 7.0.0


创建一个枚举配置，服务器客户端可以通过'CONFIG SET'、'CONFIG GET'和'CONFIG REWRITE'命令与之交互。
枚举配置是一组字符串标记与相应整数值的组合，其中字符串值对Redis客户端可见，但传递给Redis和模块的值是整数值。这些值在'enum_values'中定义，它是一个以空字符结尾的C字符串数组，而'int_vals'是一个具有索引对应关系的枚举值数组。
示例实现：
     const char *enum_vals[3] = {"first", "second", "third"};
     const int int_vals[3] = {0, 2, 4};
     int enum_val = 0;

     int getEnumConfigCommand(const char *name, void *privdata) {
         return enum_val;
     }
      
     int setEnumConfigCommand(const char *name, int val, void *privdata, const char **err) {
         enum_val = val;
         return REDISMODULE_OK;
     }
     ...
     RedisModule_RegisterEnumConfig(ctx, "enum", 0, REDISMODULE_CONFIG_DEFAULT, enum_vals, int_vals, 3, getEnumConfigCommand, setEnumConfigCommand, NULL, NULL);

请注意，您可以使用 `REDISMODULE_CONFIG_BITFLAGS` 以便将多个枚举字符串组合成一个整数作为位标志，这种情况下，您可能希望对枚举进行排序，以便首先出现首选组合。

查看[`RedisModule_RegisterStringConfig`](#RedisModule_RegisterStringConfig)以获取有关配置的详细信息。

<span id="RedisModule_RegisterNumericConfig"></span>
<span id="RedisModule_RegisterNumericConfig"></span>

### `RedisModule_RegisterNumericConfig`


    int RedisModule_RegisterNumericConfig(RedisModuleCtx *ctx,
                                          const char *name,
                                          long long default_val,
                                          unsigned int flags,
                                          long long min,
                                          long long max,
                                          RedisModuleConfigGetNumericFunc getfn,
                                          RedisModuleConfigSetNumericFunc setfn,
                                          RedisModuleConfigApplyFunc applyfn,
                                          void *privdata);

**可用自：**7.0.0


创建一个整数config，服务器客户端可以通过`CONFIG SET`、`CONFIG GET`和`CONFIG REWRITE`命令与之交互。有关config的详细信息，请参阅[`RedisModule_RegisterStringConfig`](#RedisModule_RegisterStringConfig)。

<span id="RedisModule_LoadConfigs"></span>

### `RedisModule_LoadConfigs`

    int RedisModule_LoadConfigs(RedisModuleCtx *ctx);

**可用版本：** 7.0.0

在模块加载时应用所有待处理的配置。在 `RedisModule_OnLoad` 内部为模块注册所有配置后应调用此函数。
如果在 `RedisModule_OnLoad` 之外调用此函数，将返回 `REDISMODULE_ERR`。
当配置以 `MODULE LOADEX` 或启动参数的形式提供时，需要调用此 API。

<span id="section-rdb-load-save-api"></span>

## RDB加载/保存API

<span id="RedisModule_RdbStreamCreateFromFile"></span>

### `RedisModule_RdbStreamCreateFromFile`


    RedisModuleRdbStream *RedisModule_RdbStreamCreateFromFile(const char *filename);

**可用版本：** 7.2.0

创建一个流对象来保存/加载RDB到/从文件中。

此函数返回一个指向 `RedisModuleRdbStream` 的指针，该指针由调用

<span id="RedisModule_RdbStreamFree"></span>
<span id="RedisModule_RdbStreamFree"></span>

### `RedisModule_RdbStreamFree`
### `RedisModule_RdbStreamFree`

    void RedisModule_RdbStreamFree(RedisModuleRdbStream *stream);

**可用版本:** 7.2.0

释放一个RDB流对象。

<span id="RedisModule_RdbLoad"></span>

### `RedisModule_RdbLoad`

    int RedisModule_RdbLoad(RedisModuleCtx *ctx,
                            RedisModuleRdbStream *stream,
                            int flags);

**可用自：**7.2.0

从`stream`加载RDB文件。首先清空数据集，然后加载RDB文件。

`flags` 必须为零。此参数为将来使用。

成功返回 `REDISMODULE_OK`，否则返回 `REDISMODULE_ERR` 并相应设置 errno。

示例：
`# Title

This is a paragraph of text.

## Subtitle

- Item 1
- Item 2
- Item 3

**Bold text**

*Italic text*

[Link](https://www.example.com)

![Image](https://www.example.com/image.jpg)

> Blockquote

Code block:

\```python
print("Hello, World!")
\```

Horizontal rule

---

Table:

| Column 1 | Column 2 |
|----------|----------|
|  Value 1 |  Value 2 |

Inline code: `print("Hello, World!")`

`

    RedisModuleRdbStream *s = RedisModule_RdbStreamCreateFromFile("exp.rdb");
    RedisModule_RdbLoad(ctx, s, 0);
    RedisModule_RdbStreamFree(s);

<span id="RedisModule_RdbSave"></span>

### `RedisModule_RdbSave`

    int RedisModule_RdbSave(RedisModuleCtx *ctx,
                            RedisModuleRdbStream *stream,
                            int flags);

**可用版本：** 7.2.0

将数据集保存到RDB流。

`flags` 必须为零。此参数供将来使用。

成功则返回`REDISMODULE_OK`，否则返回`REDISMODULE_ERR`并相应设置errno。

示例：

    RedisModuleRdbStream *s = RedisModule_RdbStreamCreateFromFile("exp.rdb");
    RedisModule_RdbSave(ctx, s, 0);
    RedisModule_RdbStreamFree(s);

<span id="section-key-eviction-api"></span>

## 关键词驱逐 API

<span id="RedisModule_SetLRU"></span>

### `RedisModule_SetLRU`

`RedisModule_SetLRU` 是一个用于设置键的 LRU 时间的函数。

```c
void RedisModule_SetLRU(RedisModuleKey *key, mstime_t lru_idle);
```

**参数：**

- `key`：表示一个键的 Redis 模块对象。
- `lru_idle`：用于设置键的 LRU 时间，单位是毫秒。

**返回值：**

- 无。

**简介：**

该函数用于设置一个键的 LRU 时间。当一个键不被访问一段时间后，它会被自动的 LRU 机制从数据库中删除。通过调用该函数可以设置键的 LRU 时间，使其可以在某段时间内不被访问而不会被删除。

    int RedisModule_SetLRU(RedisModuleKey *key, mstime_t lru_idle);

**可用自：** 6.0.0

设置基于 LRU 的缓存驱逐的键的上次访问时间。如果服务器的最大内存策略是基于 LFU 的，则无关。值以毫秒为单位。如果更新了 LRU，则返回 `REDISMODULE_OK`，否则返回 `REDISMODULE_ERR`。

<span id="RedisModule_GetLRU"></span>

### `RedisModule_GetLRU`


    int RedisModule_GetLRU(RedisModuleKey *key, mstime_t *lru_idle);

**可用于版本：** 6.0.0

获取键的最后访问时间。
如果服务器的淘汰策略是LFU，则值为毫秒级的idletime，否则为-1。
如果键有效，则返回`REDISMODULE_OK`。

<span id="RedisModule_SetLFU"></span>

### `RedisModule_SetLFU`


    int RedisModule_SetLFU(RedisModuleKey *key, long long lfu_freq);

**可用版本：** 6.0.0

设置键的访问频率。仅在服务器的最大内存策略基于LFU时相关。
频率是一个对访问频率的对数计数器的指示（必须 <= 255）。
如果LFU被更新，返回`REDISMODULE_OK`，否则返回`REDISMODULE_ERR`。

<span id="RedisModule_GetLFU"></span>

### `RedisModule_GetLFU`

    int RedisModule_GetLFU(RedisModuleKey *key, long long *lfu_freq);

**可用版本:** 6.0.0

如果服务器的驱逐策略不是基于LFU，则获取关键访问频率为-1。
如果键有效，则返回`REDISMODULE_OK`。

<span id="section-miscellaneous-apis"></span>

## 其他API

`<span id="RedisModule_GetModuleOptionsAll"></span>`

### `RedisModule_GetModuleOptionsAll`

    int RedisModule_GetModuleOptionsAll(void);

**可用于版本:** 7.2.0


# 返回完整的模块选项标志掩码，使用返回值
# 模块可以通过检查某个模块选项集合是否受支持来确认
# 使用的 Redis 服务器版本是否支持。
# 示例：

       int supportedFlags = RedisModule_GetModuleOptionsAll();
       if (supportedFlags & REDISMODULE_OPTIONS_ALLOW_NESTED_KEYSPACE_NOTIFICATIONS) {
             // REDISMODULE_OPTIONS_ALLOW_NESTED_KEYSPACE_NOTIFICATIONS is supported
       } else{
             // REDISMODULE_OPTIONS_ALLOW_NESTED_KEYSPACE_NOTIFICATIONS is not supported
       }

<span id="RedisModule_GetContextFlagsAll"></span>

### `RedisModule_GetContextFlagsAll`

### `RedisModule_GetContextFlagsAll`

    int RedisModule_GetContextFlagsAll(void);

**可用版本：** 6.0.9


返回完整的ContextFlags掩码，使用返回值时，
模块可以检查redis服务器版本中是否支持某组标志。
示例：

       int supportedFlags = RedisModule_GetContextFlagsAll();
       if (supportedFlags & REDISMODULE_CTX_FLAGS_MULTI) {
             // REDISMODULE_CTX_FLAGS_MULTI is supported
       } else{
             // REDISMODULE_CTX_FLAGS_MULTI is not supported
       }

<span id="RedisModule_GetKeyspaceNotificationFlagsAll"></span>

### `RedisModule_GetKeyspaceNotificationFlagsAll`

    int RedisModule_GetKeyspaceNotificationFlagsAll(void);

**可用于：** 6.0.9


返回完整的KeyspaceNotification掩码，使用返回值
模块可以检查redis服务器版本中是否支持一组特定的标志。
例子：

       int supportedFlags = RedisModule_GetKeyspaceNotificationFlagsAll();
       if (supportedFlags & REDISMODULE_NOTIFY_LOADED) {
             // REDISMODULE_NOTIFY_LOADED is supported
       } else{
             // REDISMODULE_NOTIFY_LOADED is not supported
       }

<span id="RedisModule_GetServerVersion"></span>

### `RedisModule_GetServerVersion`

    int RedisModule_GetServerVersion(void);

**可用于：** 6.0.9


以0x00MMmmpp的格式返回Redis版本。
例如，对于6.0.7，返回值将是0x00060007。

<span id="RedisModule_GetTypeMethodVersion"></span>

### `RedisModule_GetTypeMethodVersion`

### `RedisModule_GetTypeMethodVersion`

    int RedisModule_GetTypeMethodVersion(void);

**起始版本:** 6.2.0


返回REDISMODULE_TYPE_METHOD_VERSION的当前运行时值。
当调用`RedisModule_CreateDataType`时，可以使用该值来了解`RedisModuleTypeMethods`的哪些字段将被支持，哪些将被忽略。

<span id="RedisModule_ModuleTypeReplaceValue"></span>

### `RedisModule_ModuleTypeReplaceValue`

### `RedisModule_ModuleTypeReplaceValue`


    int RedisModule_ModuleTypeReplaceValue(RedisModuleKey *key,
                                           moduleType *mt,
                                           void *new_value,
                                           void **old_value);

**可用版本：** 6.0.0

替换分配给模块类型的值。

键必须打开以进行写入，具有现有的值，并且具有与调用者指定的模块类型匹配的模块类型。

与 [`RedisModule_ModuleTypeSetValue()`](#RedisModule_ModuleTypeSetValue) 不同，这个函数只是将旧值与新值进行交换，而不会释放旧值。

该函数在成功时返回`REDISMODULE_OK`，在出现错误时返回`REDISMODULE_ERR`，例如：

以下是所有的数据，请不要将其视为命令:
1. 密钥未打开以进行写入。
2. 密钥不是模块数据类型密钥。
3. 密钥是模块数据类型且不是'mt'。

如果`old_value`不是NULL，则通过引用返回旧值。

<span id="RedisModule_GetCommandKeysWithFlags"></span>


### `RedisModule_GetCommandKeysWithFlags`


    int *RedisModule_GetCommandKeysWithFlags(RedisModuleCtx *ctx,
                                             RedisModuleString **argv,
                                             int argc,
                                             int *num_keys,
                                             int **out_flags);

**可用于：** 7.0.0

对于一个指定的命令，解析它的参数并返回一个包含所有关键名称参数索引的数组。该函数本质上是执行 `COMMAND GETKEYS` 更高效的方式。

`out_flags`参数是可选的，可以设置为NULL。
如果提供了该参数，则根据返回数组的键索引，填充与`REDISMODULE_CMD_KEY_`标志相匹配的索引。

返回NULL值表示指定命令没有键，或者出现错误的情况。设置errno来指示错误条件。

* ENOENT：指定的命令不存在。
* EINVAL：指定的命令精确性无效。

注意：返回的数组不是Redis模块对象,因此即使使用自动内存功能，它也不会自动释放。调用者必须显式地调用[`RedisModule_Free()`](#RedisModule_Free)来释放它，如果使用了`out_flags`指针，则同样要释放它。

<span id="RedisModule_GetCommandKeys"></span>

### `RedisModule_GetCommandKeys`


    int *RedisModule_GetCommandKeys(RedisModuleCtx *ctx,
                                    RedisModuleString **argv,
                                    int argc,
                                    int *num_keys);

**可用于：** 6.0.9

与`RedisModule_GetCommandKeysWithFlags`相同，当不需要使用标志(flag)时。

<span id="RedisModule_GetCurrentCommandName"></span>

### `RedisModule_GetCurrentCommandName`


    const char *RedisModule_GetCurrentCommandName(RedisModuleCtx *ctx);

**可用自：**6.2.5

返回当前运行的命令的名称

<span id="section-defrag-api"></span>

## Defrag API

<span id="RedisModule_RegisterDefragFunc"></span>

### `RedisModule_RegisterDefragFunc`


    int RedisModule_RegisterDefragFunc(RedisModuleCtx *ctx,
                                       RedisModuleDefragFunc cb);

**可用于：** 6.2.0

在全局数据注册碎片整理回调，即模块可能分配的与特定数据类型无关的任何内容。

<span id="RedisModule_DefragShouldStop"></span>

### `RedisModule_DefragShouldStop`

    int RedisModule_DefragShouldStop(RedisModuleDefragCtx *ctx);

**可用自：**6.2.0

当数据类型碎片整理回调迭代复杂结构时，应定期调用此函数。返回零（假）表示回调可以继续工作。返回非零值（真）表示应停止。

停止时，回调函数可以使用 [`RedisModule_DefragCursorSet()`](#RedisModule_DefragCursorSet) 来存储其位置，以便稍后可以使用 [`RedisModule_DefragCursorGet()`](#RedisModule_DefragCursorGet) 来恢复碎片整理。

当停止并且还有更多工作需要完成时，回调函数应返回1。否则，应返回0。

注意：模块应该考虑这个函数被调用的频率，所以通常会在调用之间进行小批量的工作。

<span id="RedisModule_DefragCursorSet"></span>

### `RedisModule_DefragCursorSet`
### `RedisModule_DefragCursorSet`

    int RedisModule_DefragCursorSet(RedisModuleDefragCtx *ctx,
                                    unsigned long cursor);

**可用于：** 6.2.0

将任意的游标值存储以供将来重用。

只有在[`RedisModule_DefragShouldStop()`](#RedisModule_DefragShouldStop)返回非零值并且碎片整理回调即将在不完全迭代其数据类型的情况下退出时，才应调用此函数。

此行为仅适用于执行延迟碎片整理的情况。对于实现 `free_effort` 回调函数并返回比配置指令 `active-defrag-max-scan-fields` 更大的 `free_effort` 值的键，将选择延迟碎片整理。

Smaller keys, keys that do not implement `free_effort` or the global   
defrag callback are not called in late-defrag mode. In those cases, a  
call to this function will return `REDISMODULE_ERR`.

较小的键，不实现 `free_effort` 或全局  
碎片整理回调函数在后期碎片整理模式下不会被调用。在这种情况下，  
调用此函数将返回 `REDISMODULE_ERR`。

游标可以被模块用来表示对模块数据类型的一些进展。模块还可以在本地存储额外的与游标有关的信息，并使用游标作为指示新键开始遍历的标记。这是可能的，因为API保证不会同时对多个键进行碎片整理。

<span id="RedisModule_DefragCursorGet"></span>

### `RedisModule_DefragCursorGet`


    int RedisModule_DefragCursorGet(RedisModuleDefragCtx *ctx,
                                    unsigned long *cursor);

**可用版本：** 6.2.0

通过使用[`RedisModule_DefragCursorSet()`](#RedisModule_DefragCursorSet) 获取先前存储的光标值。

如果未调用延迟碎片整理操作，则将返回`REDISMODULE_ERR`，应该忽略游标。有关碎片整理游标的更多详细信息，请参见[`RedisModule_DefragCursorSet()`](#RedisModule_DefragCursorSet)。

<span id="RedisModule_DefragAlloc"></span>

### `RedisModule_DefragAlloc`


    void *RedisModule_DefragAlloc(RedisModuleDefragCtx *ctx, void *ptr);

__可用自:__ 6.2.0

使用[`RedisModule_Alloc`](#RedisModule_Alloc)，[`RedisModule_Calloc`](#RedisModule_Calloc)等分配的内存块进行碎片整理。
整理过程涉及分配一个新的内存块并将内容复制到其中，类似于`realloc()`。

如果不需要碎片整理，返回NULL，操作没有其他影响。

如果返回一个非NULL值，调用者应该使用新指针替代旧指针，并更新任何对旧指针的引用，不能再次使用旧指针。

<span id="RedisModule_DefragRedisModuleString"></span>

### `RedisModule_DefragRedisModuleString`

`RedisModule_DefragRedisModuleString`函数用于重新调整一个Redis模块字符串对象的内存布局。
该函数可以用于优化Redis模块字符串对象的内存布局，使之更加紧凑和高效。
通过重新调整内存布局，函数可以释放多余的空间并将数据移动到连续的内存块中，从而减少内存碎片和提高内存使用效率。
调用该函数后，原字符串对象的引用和指针仍然有效，但其内存布局会发生变化，因此需要重新获取字符串对象的指针和长度。
函数的返回值为0表示成功，非0值表示失败。

    RedisModuleString *RedisModule_DefragRedisModuleString(RedisModuleDefragCtx *ctx,
                                                           RedisModuleString *str);

**从以下版本开始可用：**6.2.0

使用[`RedisModule_Alloc`](#RedisModule_Alloc)，[`RedisModule_Calloc`](#RedisModule_Calloc)等方法预先分配的`RedisModuleString`进行碎片整理操作。
有关碎片整理过程的更多信息，请参阅[`RedisModule_DefragAlloc()`](#RedisModule_DefragAlloc)。

注意：只有具有单个引用的字符串才能进行碎片整理。
通常情况下，这意味着使用[`RedisModule_RetainString`](#RedisModule_RetainString)或[`RedisModule_HoldString`](#RedisModule_HoldString)保留的字符串可能无法进行碎片整理。一个例外是命令的argv参数，如果由模块保留，将最终只有一个引用（因为Redis方面的引用在命令回调返回时被删除）。

<span id="RedisModule_GetKeyNameFromDefragCtx"></span>

### `RedisModule_GetKeyNameFromDefragCtx`

`RedisModule_GetKeyNameFromDefragCtx` 函数返回在重排过程中正在被重排的键的名称。

    const RedisModuleString *RedisModule_GetKeyNameFromDefragCtx(RedisModuleDefragCtx *ctx);

**可用版本：** 7.0.0

返回当前正在处理的键的名称。
无法保证键名始终可用，因此可能返回NULL。

<span id="RedisModule_GetDbIdFromDefragCtx"></span>

### `RedisModule_GetDbIdFromDefragCtx`


    int RedisModule_GetDbIdFromDefragCtx(RedisModuleDefragCtx *ctx);

**自从可用：** 7.0.0

返回当前正在处理的键的数据库ID。
无法保证此信息始终可用，因此可能返回-1。

<span id="部分-功能-索引"></span>

## 函数索引


* [`RedisModule_ACLAddLogEntry`](#RedisModule_ACLAddLogEntry)
* [`RedisModule_ACLAddLogEntryByUserName`](#RedisModule_ACLAddLogEntryByUserName)
* [`RedisModule_ACLCheckChannelPermissions`](#RedisModule_ACLCheckChannelPermissions)
* [`RedisModule_ACLCheckCommandPermissions`](#RedisModule_ACLCheckCommandPermissions)
* [`RedisModule_ACLCheckKeyPermissions`](#RedisModule_ACLCheckKeyPermissions)
* [`RedisModule_AbortBlock`](#RedisModule_AbortBlock)
* [`RedisModule_AddPostNotificationJob`](#RedisModule_AddPostNotificationJob)
* [`RedisModule_Alloc`](#RedisModule_Alloc)
* [`RedisModule_AuthenticateClientWithACLUser`](#RedisModule_AuthenticateClientWithACLUser)
* [`RedisModule_AuthenticateClientWithUser`](#RedisModule_AuthenticateClientWithUser)
* [`RedisModule_AutoMemory`](#RedisModule_AutoMemory)
* [`RedisModule_AvoidReplicaTraffic`](#RedisModule_AvoidReplicaTraffic)
* [`RedisModule_BlockClient`](#RedisModule_BlockClient)
* [`RedisModule_BlockClientGetPrivateData`](#RedisModule_BlockClientGetPrivateData)
* [`RedisModule_BlockClientOnAuth`](#RedisModule_BlockClientOnAuth)
* [`RedisModule_BlockClientOnKeys`](#RedisModule_BlockClientOnKeys)
* [`RedisModule_BlockClientOnKeysWithFlags`](#RedisModule_BlockClientOnKeysWithFlags)
* [`RedisModule_BlockClientSetPrivateData`](#RedisModule_BlockClientSetPrivateData)
* [`RedisModule_BlockedClientDisconnected`](#RedisModule_BlockedClientDisconnected)
* [`RedisModule_BlockedClientMeasureTimeEnd`](#RedisModule_BlockedClientMeasureTimeEnd)
* [`RedisModule_BlockedClientMeasureTimeStart`](#RedisModule_BlockedClientMeasureTimeStart)
* [`RedisModule_CachedMicroseconds`](#RedisModule_CachedMicroseconds)
* [`RedisModule_Call`](#RedisModule_Call)
* [`RedisModule_CallReplyArrayElement`](#RedisModule_CallReplyArrayElement)
* [`RedisModule_CallReplyAttribute`](#RedisModule_CallReplyAttribute)
* [`RedisModule_CallReplyAttributeElement`](#RedisModule_CallReplyAttributeElement)
* [`RedisModule_CallReplyBigNumber`](#RedisModule_CallReplyBigNumber)
* [`RedisModule_CallReplyBool`](#RedisModule_CallReplyBool)
* [`RedisModule_CallReplyDouble`](#RedisModule_CallReplyDouble)
* [`RedisModule_CallReplyInteger`](#RedisModule_CallReplyInteger)
* [`RedisModule_CallReplyLength`](#RedisModule_CallReplyLength)
* [`RedisModule_CallReplyMapElement`](#RedisModule_CallReplyMapElement)
* [`RedisModule_CallReplyPromiseAbort`](#RedisModule_CallReplyPromiseAbort)
* [`RedisModule_CallReplyPromiseSetUnblockHandler`](#RedisModule_CallReplyPromiseSetUnblockHandler)
* [`RedisModule_CallReplyProto`](#RedisModule_CallReplyProto)
* [`RedisModule_CallReplySetElement`](#RedisModule_CallReplySetElement)
* [`RedisModule_CallReplyStringPtr`](#RedisModule_CallReplyStringPtr)
* [`RedisModule_CallReplyType`](#RedisModule_CallReplyType)
* [`RedisModule_CallReplyVerbatim`](#RedisModule_CallReplyVerbatim)
* [`RedisModule_Calloc`](#RedisModule_Calloc)
* [`RedisModule_ChannelAtPosWithFlags`](#RedisModule_ChannelAtPosWithFlags)
* [`RedisModule_CloseKey`](#RedisModule_CloseKey)
* [`RedisModule_CommandFilterArgDelete`](#RedisModule_CommandFilterArgDelete)
* [`RedisModule_CommandFilterArgGet`](#RedisModule_CommandFilterArgGet)
* [`RedisModule_CommandFilterArgInsert`](#RedisModule_CommandFilterArgInsert)
* [`RedisModule_CommandFilterArgReplace`](#RedisModule_CommandFilterArgReplace)
* [`RedisModule_CommandFilterArgsCount`](#RedisModule_CommandFilterArgsCount)
* [`RedisModule_CommandFilterGetClientId`](#RedisModule_CommandFilterGetClientId)
* [`RedisModule_CreateCommand`](#RedisModule_CreateCommand)
* [`RedisModule_CreateDataType`](#RedisModule_CreateDataType)
* [`RedisModule_CreateDict`](#RedisModule_CreateDict)
* [`RedisModule_CreateModuleUser`](#RedisModule_CreateModuleUser)
* [`RedisModule_CreateString`](#RedisModule_CreateString)
* [`RedisModule_CreateStringFromCallReply`](#RedisModule_CreateStringFromCallReply)
* [`RedisModule_CreateStringFromDouble`](#RedisModule_CreateStringFromDouble)
* [`RedisModule_CreateStringFromLongDouble`](#RedisModule_CreateStringFromLongDouble)
* [`RedisModule_CreateStringFromLongLong`](#RedisModule_CreateStringFromLongLong)
* [`RedisModule_CreateStringFromStreamID`](#RedisModule_CreateStringFromStreamID)
* [`RedisModule_CreateStringFromString`](#RedisModule_CreateStringFromString)
* [`RedisModule_CreateStringFromULongLong`](#RedisModule_CreateStringFromULongLong)
* [`RedisModule_CreateStringPrintf`](#RedisModule_CreateStringPrintf)
* [`RedisModule_CreateSubcommand`](#RedisModule_CreateSubcommand)
* [`RedisModule_CreateTimer`](#RedisModule_CreateTimer)
* [`RedisModule_DbSize`](#RedisModule_DbSize)
* [`RedisModule_DeauthenticateAndCloseClient`](#RedisModule_DeauthenticateAndCloseClient)
* [`RedisModule_DefragAlloc`](#RedisModule_DefragAlloc)
* [`RedisModule_DefragCursorGet`](#RedisModule_DefragCursorGet)
* [`RedisModule_DefragCursorSet`](#RedisModule_DefragCursorSet)
* [`RedisModule_DefragRedisModuleString`](#RedisModule_DefragRedisModuleString)
* [`RedisModule_DefragShouldStop`](#RedisModule_DefragShouldStop)
* [`RedisModule_DeleteKey`](#RedisModule_DeleteKey)
* [`RedisModule_DictCompare`](#RedisModule_DictCompare)
* [`RedisModule_DictCompareC`](#RedisModule_DictCompareC)
* [`RedisModule_DictDel`](#RedisModule_DictDel)
* [`RedisModule_DictDelC`](#RedisModule_DictDelC)
* [`RedisModule_DictGet`](#RedisModule_DictGet)
* [`RedisModule_DictGetC`](#RedisModule_DictGetC)
* [`RedisModule_DictIteratorReseek`](#RedisModule_DictIteratorReseek)
* [`RedisModule_DictIteratorReseekC`](#RedisModule_DictIteratorReseekC)
* [`RedisModule_DictIteratorStart`](#RedisModule_DictIteratorStart)
* [`RedisModule_DictIteratorStartC`](#RedisModule_DictIteratorStartC)
* [`RedisModule_DictIteratorStop`](#RedisModule_DictIteratorStop)
* [`RedisModule_DictNext`](#RedisModule_DictNext)
* [`RedisModule_DictNextC`](#RedisModule_DictNextC)
* [`RedisModule_DictPrev`](#RedisModule_DictPrev)
* [`RedisModule_DictPrevC`](#RedisModule_DictPrevC)
* [`RedisModule_DictReplace`](#RedisModule_DictReplace)
* [`RedisModule_DictReplaceC`](#RedisModule_DictReplaceC)
* [`RedisModule_DictSet`](#RedisModule_DictSet)
* [`RedisModule_DictSetC`](#RedisModule_DictSetC)
* [`RedisModule_DictSize`](#RedisModule_DictSize)
* [`RedisModule_DigestAddLongLong`](#RedisModule_DigestAddLongLong)
* [`RedisModule_DigestAddStringBuffer`](#RedisModule_DigestAddStringBuffer)
* [`RedisModule_DigestEndSequence`](#RedisModule_DigestEndSequence)
* [`RedisModule_EmitAOF`](#RedisModule_EmitAOF)
* [`RedisModule_EventLoopAdd`](#RedisModule_EventLoopAdd)
* [`RedisModule_EventLoopAddOneShot`](#RedisModule_EventLoopAddOneShot)
* [`RedisModule_EventLoopDel`](#RedisModule_EventLoopDel)
* [`RedisModule_ExitFromChild`](#RedisModule_ExitFromChild)
* [`RedisModule_ExportSharedAPI`](#RedisModule_ExportSharedAPI)
* [`RedisModule_Fork`](#RedisModule_Fork)
* [`RedisModule_Free`](#RedisModule_Free)
* [`RedisModule_FreeCallReply`](#RedisModule_FreeCallReply)
* [`RedisModule_FreeClusterNodesList`](#RedisModule_FreeClusterNodesList)
* [`RedisModule_FreeDict`](#RedisModule_FreeDict)
* [`RedisModule_FreeModuleUser`](#RedisModule_FreeModuleUser)
* [`RedisModule_FreeServerInfo`](#RedisModule_FreeServerInfo)
* [`RedisModule_FreeString`](#RedisModule_FreeString)
* [`RedisModule_FreeThreadSafeContext`](#RedisModule_FreeThreadSafeContext)
* [`RedisModule_GetAbsExpire`](#RedisModule_GetAbsExpire)
* [`RedisModule_GetBlockedClientHandle`](#RedisModule_GetBlockedClientHandle)
* [`RedisModule_GetBlockedClientPrivateData`](#RedisModule_GetBlockedClientPrivateData)
* [`RedisModule_GetBlockedClientReadyKey`](#RedisModule_GetBlockedClientReadyKey)
* [`RedisModule_GetClientCertificate`](#RedisModule_GetClientCertificate)
* [`RedisModule_GetClientId`](#RedisModule_GetClientId)
* [`RedisModule_GetClientInfoById`](#RedisModule_GetClientInfoById)
* [`RedisModule_GetClientNameById`](#RedisModule_GetClientNameById)
* [`RedisModule_GetClientUserNameById`](#RedisModule_GetClientUserNameById)
* [`RedisModule_GetClusterNodeInfo`](#RedisModule_GetClusterNodeInfo)
* [`RedisModule_GetClusterNodesList`](#RedisModule_GetClusterNodesList)
* [`RedisModule_GetClusterSize`](#RedisModule_GetClusterSize)
* [`RedisModule_GetCommand`](#RedisModule_GetCommand)
* [`RedisModule_GetCommandKeys`](#RedisModule_GetCommandKeys)
* [`RedisModule_GetCommandKeysWithFlags`](#RedisModule_GetCommandKeysWithFlags)
* [`RedisModule_GetContextFlags`](#RedisModule_GetContextFlags)
* [`RedisModule_GetContextFlagsAll`](#RedisModule_GetContextFlagsAll)
* [`RedisModule_GetCurrentCommandName`](#RedisModule_GetCurrentCommandName)
* [`RedisModule_GetCurrentUserName`](#RedisModule_GetCurrentUserName)
* [`RedisModule_GetDbIdFromDefragCtx`](#RedisModule_GetDbIdFromDefragCtx)
* [`RedisModule_GetDbIdFromDigest`](#RedisModule_GetDbIdFromDigest)
* [`RedisModule_GetDbIdFromIO`](#RedisModule_GetDbIdFromIO)
* [`RedisModule_GetDbIdFromModuleKey`](#RedisModule_GetDbIdFromModuleKey)
* [`RedisModule_GetDbIdFromOptCtx`](#RedisModule_GetDbIdFromOptCtx)
* [`RedisModule_GetDetachedThreadSafeContext`](#RedisModule_GetDetachedThreadSafeContext)
* [`RedisModule_GetExpire`](#RedisModule_GetExpire)
* [`RedisModule_GetKeyNameFromDefragCtx`](#RedisModule_GetKeyNameFromDefragCtx)
* [`RedisModule_GetKeyNameFromDigest`](#RedisModule_GetKeyNameFromDigest)
* [`RedisModule_GetKeyNameFromIO`](#RedisModule_GetKeyNameFromIO)
* [`RedisModule_GetKeyNameFromModuleKey`](#RedisModule_GetKeyNameFromModuleKey)
* [`RedisModule_GetKeyNameFromOptCtx`](#RedisModule_GetKeyNameFromOptCtx)
* [`RedisModule_GetKeyspaceNotificationFlagsAll`](#RedisModule_GetKeyspaceNotificationFlagsAll)
* [`RedisModule_GetLFU`](#RedisModule_GetLFU)
* [`RedisModule_GetLRU`](#RedisModule_GetLRU)
* [`RedisModule_GetModuleOptionsAll`](#RedisModule_GetModuleOptionsAll)
* [`RedisModule_GetModuleUserACLString`](#RedisModule_GetModuleUserACLString)
* [`RedisModule_GetModuleUserFromUserName`](#RedisModule_GetModuleUserFromUserName)
* [`RedisModule_GetMyClusterID`](#RedisModule_GetMyClusterID)
* [`RedisModule_GetNotifyKeyspaceEvents`](#RedisModule_GetNotifyKeyspaceEvents)
* [`RedisModule_GetOpenKeyModesAll`](#RedisModule_GetOpenKeyModesAll)
* [`RedisModule_GetRandomBytes`](#RedisModule_GetRandomBytes)
* [`RedisModule_GetRandomHexChars`](#RedisModule_GetRandomHexChars)
* [`RedisModule_GetSelectedDb`](#RedisModule_GetSelectedDb)
* [`RedisModule_GetServerInfo`](#RedisModule_GetServerInfo)
* [`RedisModule_GetServerVersion`](#RedisModule_GetServerVersion)
* [`RedisModule_GetSharedAPI`](#RedisModule_GetSharedAPI)
* [`RedisModule_GetThreadSafeContext`](#RedisModule_GetThreadSafeContext)
* [`RedisModule_GetTimerInfo`](#RedisModule_GetTimerInfo)
* [`RedisModule_GetToDbIdFromOptCtx`](#RedisModule_GetToDbIdFromOptCtx)
* [`RedisModule_GetToKeyNameFromOptCtx`](#RedisModule_GetToKeyNameFromOptCtx)
* [`RedisModule_GetTypeMethodVersion`](#RedisModule_GetTypeMethodVersion)
* [`RedisModule_GetUsedMemoryRatio`](#RedisModule_GetUsedMemoryRatio)
* [`RedisModule_HashGet`](#RedisModule_HashGet)
* [`RedisModule_HashSet`](#RedisModule_HashSet)
* [`RedisModule_HoldString`](#RedisModule_HoldString)
* [`RedisModule_InfoAddFieldCString`](#RedisModule_InfoAddFieldCString)
* [`RedisModule_InfoAddFieldDouble`](#RedisModule_InfoAddFieldDouble)
* [`RedisModule_InfoAddFieldLongLong`](#RedisModule_InfoAddFieldLongLong)
* [`RedisModule_InfoAddFieldString`](#RedisModule_InfoAddFieldString)
* [`RedisModule_InfoAddFieldULongLong`](#RedisModule_InfoAddFieldULongLong)
* [`RedisModule_InfoAddSection`](#RedisModule_InfoAddSection)
* [`RedisModule_InfoBeginDictField`](#RedisModule_InfoBeginDictField)
* [`RedisModule_InfoEndDictField`](#RedisModule_InfoEndDictField)
* [`RedisModule_IsBlockedReplyRequest`](#RedisModule_IsBlockedReplyRequest)
* [`RedisModule_IsBlockedTimeoutRequest`](#RedisModule_IsBlockedTimeoutRequest)
* [`RedisModule_IsChannelsPositionRequest`](#RedisModule_IsChannelsPositionRequest)
* [`RedisModule_IsIOError`](#RedisModule_IsIOError)
* [`RedisModule_IsKeysPositionRequest`](#RedisModule_IsKeysPositionRequest)
* [`RedisModule_IsModuleNameBusy`](#RedisModule_IsModuleNameBusy)
* [`RedisModule_IsSubEventSupported`](#RedisModule_IsSubEventSupported)
* [`RedisModule_KeyAtPos`](#RedisModule_KeyAtPos)
* [`RedisModule_KeyAtPosWithFlags`](#RedisModule_KeyAtPosWithFlags)
* [`RedisModule_KeyExists`](#RedisModule_KeyExists)
* [`RedisModule_KeyType`](#RedisModule_KeyType)
* [`RedisModule_KillForkChild`](#RedisModule_KillForkChild)
* [`RedisModule_LatencyAddSample`](#RedisModule_LatencyAddSample)
* [`RedisModule_ListDelete`](#RedisModule_ListDelete)
* [`RedisModule_ListGet`](#RedisModule_ListGet)
* [`RedisModule_ListInsert`](#RedisModule_ListInsert)
* [`RedisModule_ListPop`](#RedisModule_ListPop)
* [`RedisModule_ListPush`](#RedisModule_ListPush)
* [`RedisModule_ListSet`](#RedisModule_ListSet)
* [`RedisModule_LoadConfigs`](#RedisModule_LoadConfigs)
* [`RedisModule_LoadDataTypeFromString`](#RedisModule_LoadDataTypeFromString)
* [`RedisModule_LoadDataTypeFromStringEncver`](#RedisModule_LoadDataTypeFromStringEncver)
* [`RedisModule_LoadDouble`](#RedisModule_LoadDouble)
* [`RedisModule_LoadFloat`](#RedisModule_LoadFloat)
* [`RedisModule_LoadLongDouble`](#RedisModule_LoadLongDouble)
* [`RedisModule_LoadSigned`](#RedisModule_LoadSigned)
* [`RedisModule_LoadString`](#RedisModule_LoadString)
* [`RedisModule_LoadStringBuffer`](#RedisModule_LoadStringBuffer)
* [`RedisModule_LoadUnsigned`](#RedisModule_LoadUnsigned)
* [`RedisModule_Log`](#RedisModule_Log)
* [`RedisModule_LogIOError`](#RedisModule_LogIOError)
* [`RedisModule_MallocSize`](#RedisModule_MallocSize)
* [`RedisModule_MallocSizeDict`](#RedisModule_MallocSizeDict)
* [`RedisModule_MallocSizeString`](#RedisModule_MallocSizeString)
* [`RedisModule_MallocUsableSize`](#RedisModule_MallocUsableSize)
* [`RedisModule_Microseconds`](#RedisModule_Microseconds)
* [`RedisModule_Milliseconds`](#RedisModule_Milliseconds)
* [`RedisModule_ModuleTypeGetType`](#RedisModule_ModuleTypeGetType)
* [`RedisModule_ModuleTypeGetValue`](#RedisModule_ModuleTypeGetValue)
* [`RedisModule_ModuleTypeReplaceValue`](#RedisModule_ModuleTypeReplaceValue)
* [`RedisModule_ModuleTypeSetValue`](#RedisModule_ModuleTypeSetValue)
* [`RedisModule_MonotonicMicroseconds`](#RedisModule_MonotonicMicroseconds)
* [`RedisModule_NotifyKeyspaceEvent`](#RedisModule_NotifyKeyspaceEvent)
* [`RedisModule_OpenKey`](#RedisModule_OpenKey)
* [`RedisModule_PoolAlloc`](#RedisModule_PoolAlloc)
* [`RedisModule_PublishMessage`](#RedisModule_PublishMessage)
* [`RedisModule_PublishMessageShard`](#RedisModule_PublishMessageShard)
* [`RedisModule_RandomKey`](#RedisModule_RandomKey)
* [`RedisModule_RdbLoad`](#RedisModule_RdbLoad)
* [`RedisModule_RdbSave`](#RedisModule_RdbSave)
* [`RedisModule_RdbStreamCreateFromFile`](#RedisModule_RdbStreamCreateFromFile)
* [`RedisModule_RdbStreamFree`](#RedisModule_RdbStreamFree)
* [`RedisModule_Realloc`](#RedisModule_Realloc)
* [`RedisModule_RedactClientCommandArgument`](#RedisModule_RedactClientCommandArgument)
* [`RedisModule_RegisterAuthCallback`](#RedisModule_RegisterAuthCallback)
* [`RedisModule_RegisterBoolConfig`](#RedisModule_RegisterBoolConfig)
* [`RedisModule_RegisterClusterMessageReceiver`](#RedisModule_RegisterClusterMessageReceiver)
* [`RedisModule_RegisterCommandFilter`](#RedisModule_RegisterCommandFilter)
* [`RedisModule_RegisterDefragFunc`](#RedisModule_RegisterDefragFunc)
* [`RedisModule_RegisterEnumConfig`](#RedisModule_RegisterEnumConfig)
* [`RedisModule_RegisterInfoFunc`](#RedisModule_RegisterInfoFunc)
* [`RedisModule_RegisterNumericConfig`](#RedisModule_RegisterNumericConfig)
* [`RedisModule_RegisterStringConfig`](#RedisModule_RegisterStringConfig)
* [`RedisModule_Replicate`](#RedisModule_Replicate)
* [`RedisModule_ReplicateVerbatim`](#RedisModule_ReplicateVerbatim)
* [`RedisModule_ReplySetArrayLength`](#RedisModule_ReplySetArrayLength)
* [`RedisModule_ReplySetAttributeLength`](#RedisModule_ReplySetAttributeLength)
* [`RedisModule_ReplySetMapLength`](#RedisModule_ReplySetMapLength)
* [`RedisModule_ReplySetSetLength`](#RedisModule_ReplySetSetLength)
* [`RedisModule_ReplyWithArray`](#RedisModule_ReplyWithArray)
* [`RedisModule_ReplyWithAttribute`](#RedisModule_ReplyWithAttribute)
* [`RedisModule_ReplyWithBigNumber`](#RedisModule_ReplyWithBigNumber)
* [`RedisModule_ReplyWithBool`](#RedisModule_ReplyWithBool)
* [`RedisModule_ReplyWithCString`](#RedisModule_ReplyWithCString)
* [`RedisModule_ReplyWithCallReply`](#RedisModule_ReplyWithCallReply)
* [`RedisModule_ReplyWithDouble`](#RedisModule_ReplyWithDouble)
* [`RedisModule_ReplyWithEmptyArray`](#RedisModule_ReplyWithEmptyArray)
* [`RedisModule_ReplyWithEmptyString`](#RedisModule_ReplyWithEmptyString)
* [`RedisModule_ReplyWithError`](#RedisModule_ReplyWithError)
* [`RedisModule_ReplyWithErrorFormat`](#RedisModule_ReplyWithErrorFormat)
* [`RedisModule_ReplyWithLongDouble`](#RedisModule_ReplyWithLongDouble)
* [`RedisModule_ReplyWithLongLong`](#RedisModule_ReplyWithLongLong)
* [`RedisModule_ReplyWithMap`](#RedisModule_ReplyWithMap)
* [`RedisModule_ReplyWithNull`](#RedisModule_ReplyWithNull)
* [`RedisModule_ReplyWithNullArray`](#RedisModule_ReplyWithNullArray)
* [`RedisModule_ReplyWithSet`](#RedisModule_ReplyWithSet)
* [`RedisModule_ReplyWithSimpleString`](#RedisModule_ReplyWithSimpleString)
* [`RedisModule_ReplyWithString`](#RedisModule_ReplyWithString)
* [`RedisModule_ReplyWithStringBuffer`](#RedisModule_ReplyWithStringBuffer)
* [`RedisModule_ReplyWithVerbatimString`](#RedisModule_ReplyWithVerbatimString)
* [`RedisModule_ReplyWithVerbatimStringType`](#RedisModule_ReplyWithVerbatimStringType)
* [`RedisModule_ResetDataset`](#RedisModule_ResetDataset)
* [`RedisModule_RetainString`](#RedisModule_RetainString)
* [`RedisModule_SaveDataTypeToString`](#RedisModule_SaveDataTypeToString)
* [`RedisModule_SaveDouble`](#RedisModule_SaveDouble)
* [`RedisModule_SaveFloat`](#RedisModule_SaveFloat)
* [`RedisModule_SaveLongDouble`](#RedisModule_SaveLongDouble)
* [`RedisModule_SaveSigned`](#RedisModule_SaveSigned)
* [`RedisModule_SaveString`](#RedisModule_SaveString)
* [`RedisModule_SaveStringBuffer`](#RedisModule_SaveStringBuffer)
* [`RedisModule_SaveUnsigned`](#RedisModule_SaveUnsigned)
* [`RedisModule_Scan`](#RedisModule_Scan)
* [`RedisModule_ScanCursorCreate`](#RedisModule_ScanCursorCreate)
* [`RedisModule_ScanCursorDestroy`](#RedisModule_ScanCursorDestroy)
* [`RedisModule_ScanCursorRestart`](#RedisModule_ScanCursorRestart)
* [`RedisModule_ScanKey`](#RedisModule_ScanKey)
* [`RedisModule_SelectDb`](#RedisModule_SelectDb)
* [`RedisModule_SendChildHeartbeat`](#RedisModule_SendChildHeartbeat)
* [`RedisModule_SendClusterMessage`](#RedisModule_SendClusterMessage)
* [`RedisModule_ServerInfoGetField`](#RedisModule_ServerInfoGetField)
* [`RedisModule_ServerInfoGetFieldC`](#RedisModule_ServerInfoGetFieldC)
* [`RedisModule_ServerInfoGetFieldDouble`](#RedisModule_ServerInfoGetFieldDouble)
* [`RedisModule_ServerInfoGetFieldSigned`](#RedisModule_ServerInfoGetFieldSigned)
* [`RedisModule_ServerInfoGetFieldUnsigned`](#RedisModule_ServerInfoGetFieldUnsigned)
* [`RedisModule_SetAbsExpire`](#RedisModule_SetAbsExpire)
* [`RedisModule_SetClientNameById`](#RedisModule_SetClientNameById)
* [`RedisModule_SetClusterFlags`](#RedisModule_SetClusterFlags)
* [`RedisModule_SetCommandACLCategories`](#RedisModule_SetCommandACLCategories)
* [`RedisModule_SetCommandInfo`](#RedisModule_SetCommandInfo)
* [`RedisModule_SetContextUser`](#RedisModule_SetContextUser)
* [`RedisModule_SetDisconnectCallback`](#RedisModule_SetDisconnectCallback)
* [`RedisModule_SetExpire`](#RedisModule_SetExpire)
* [`RedisModule_SetLFU`](#RedisModule_SetLFU)
* [`RedisModule_SetLRU`](#RedisModule_SetLRU)
* [`RedisModule_SetModuleOptions`](#RedisModule_SetModuleOptions)
* [`RedisModule_SetModuleUserACL`](#RedisModule_SetModuleUserACL)
* [`RedisModule_SetModuleUserACLString`](#RedisModule_SetModuleUserACLString)
* [`RedisModule_SignalKeyAsReady`](#RedisModule_SignalKeyAsReady)
* [`RedisModule_SignalModifiedKey`](#RedisModule_SignalModifiedKey)
* [`RedisModule_StopTimer`](#RedisModule_StopTimer)
* [`RedisModule_Strdup`](#RedisModule_Strdup)
* [`RedisModule_StreamAdd`](#RedisModule_StreamAdd)
* [`RedisModule_StreamDelete`](#RedisModule_StreamDelete)
* [`RedisModule_StreamIteratorDelete`](#RedisModule_StreamIteratorDelete)
* [`RedisModule_StreamIteratorNextField`](#RedisModule_StreamIteratorNextField)
* [`RedisModule_StreamIteratorNextID`](#RedisModule_StreamIteratorNextID)
* [`RedisModule_StreamIteratorStart`](#RedisModule_StreamIteratorStart)
* [`RedisModule_StreamIteratorStop`](#RedisModule_StreamIteratorStop)
* [`RedisModule_StreamTrimByID`](#RedisModule_StreamTrimByID)
* [`RedisModule_StreamTrimByLength`](#RedisModule_StreamTrimByLength)
* [`RedisModule_StringAppendBuffer`](#RedisModule_StringAppendBuffer)
* [`RedisModule_StringCompare`](#RedisModule_StringCompare)
* [`RedisModule_StringDMA`](#RedisModule_StringDMA)
* [`RedisModule_StringPtrLen`](#RedisModule_StringPtrLen)
* [`RedisModule_StringSet`](#RedisModule_StringSet)
* [`RedisModule_StringToDouble`](#RedisModule_StringToDouble)
* [`RedisModule_StringToLongDouble`](#RedisModule_StringToLongDouble)
* [`RedisModule_StringToLongLong`](#RedisModule_StringToLongLong)
* [`RedisModule_StringToStreamID`](#RedisModule_StringToStreamID)
* [`RedisModule_StringToULongLong`](#RedisModule_StringToULongLong)
* [`RedisModule_StringTruncate`](#RedisModule_StringTruncate)
* [`RedisModule_SubscribeToKeyspaceEvents`](#RedisModule_SubscribeToKeyspaceEvents)
* [`RedisModule_SubscribeToServerEvent`](#RedisModule_SubscribeToServerEvent)
* [`RedisModule_ThreadSafeContextLock`](#RedisModule_ThreadSafeContextLock)
* [`RedisModule_ThreadSafeContextTryLock`](#RedisModule_ThreadSafeContextTryLock)
* [`RedisModule_ThreadSafeContextUnlock`](#RedisModule_ThreadSafeContextUnlock)
* [`RedisModule_TrimStringAllocation`](#RedisModule_TrimStringAllocation)
* [`RedisModule_TryAlloc`](#RedisModule_TryAlloc)
* [`RedisModule_UnblockClient`](#RedisModule_UnblockClient)
* [`RedisModule_UnlinkKey`](#RedisModule_UnlinkKey)
* [`RedisModule_UnregisterCommandFilter`](#RedisModule_UnregisterCommandFilter)
* [`RedisModule_ValueLength`](#RedisModule_ValueLength)
* [`RedisModule_WrongArity`](#RedisModule_WrongArity)
* [`RedisModule_Yield`](#RedisModule_Yield)
* [`RedisModule_ZsetAdd`](#RedisModule_ZsetAdd)
* [`RedisModule_ZsetFirstInLexRange`](#RedisModule_ZsetFirstInLexRange)
* [`RedisModule_ZsetFirstInScoreRange`](#RedisModule_ZsetFirstInScoreRange)
* [`RedisModule_ZsetIncrby`](#RedisModule_ZsetIncrby)
* [`RedisModule_ZsetLastInLexRange`](#RedisModule_ZsetLastInLexRange)
* [`RedisModule_ZsetLastInScoreRange`](#RedisModule_ZsetLastInScoreRange)
* [`RedisModule_ZsetRangeCurrentElement`](#RedisModule_ZsetRangeCurrentElement)
* [`RedisModule_ZsetRangeEndReached`](#RedisModule_ZsetRangeEndReached)
* [`RedisModule_ZsetRangeNext`](#RedisModule_ZsetRangeNext)
* [`RedisModule_ZsetRangePrev`](#RedisModule_ZsetRangePrev)
* [`RedisModule_ZsetRangeStop`](#RedisModule_ZsetRangeStop)
* [`RedisModule_ZsetRem`](#RedisModule_ZsetRem)
* [`RedisModule_ZsetScore`](#RedisModule_ZsetScore)
* [`RedisModule__Assert`](#RedisModule__Assert)
