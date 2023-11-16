返回一个包含有关每个Redis命令的详细信息的数组。

`COMMAND` 命令是一种反射命令。
其回复描述了服务器能够处理的所有命令。
Redis 客户端可以在握手期间调用它以获取服务器的运行时能力。

`COMMAND`也有几个子命令。
有关详细信息，请参考其子命令。

**集群注释:**
该命令对于集群感知客户端尤为有益。
这种客户端必须识别命令中的键名以将请求路由到正确的分片。
尽管大多数命令接受一个键作为第一个参数，但有很多例外。
您可以调用"COMMAND"，然后在客户端中缓存命令及其相应的键规范规则之间的映射关系。

它返回的回复是一个数组，每个命令对应一个元素。
每个描述Redis命令的元素本身也被表示为一个数组。

命令的数组由固定数量的元素组成。
数组中的元素数量取决于服务器的版本。

1. 名称
1. 参数个数
1. 标志
1. 第一个键
1. 最后一个键
1. 步长
1. [ACL 分类][ta] (适用于 Redis 6.0)
1. [提示][tb] (适用于 Redis 7.0)
1. [键规范][td] (适用于 Redis 7.0)
1. 子命令 (适用于 Redis 7.0)

## 名称

这是命令的名称，以小写形式。

**注意：**
Redis 命令名称是不区分大小写的。

## 个参数

Arity是命令期望的参数数量。
它遵循一个简单的模式：

* 正整数表示固定数量的参数。
* 负整数表示最小数量的参数。

命令的arity（参数个数）总是包括命令本身的名称（以及适用时的子命令）。

示例：

* `GET` 的 arity 是 _2_，因为该命令只接受一个参数，并且格式始终为 `GET _key_`。
* `MGET` 的 arity 是 _-2_，因为该命令至少接受一个参数，但可能接受多个参数：`MGET _key1_ [key2] [key3] ...`。

## 标志

命令标志是一个数组。它可以包含以下简单的字符串（状态回复）：

* **admin：**该命令是管理命令。
* **asking：**该命令即使在哈希槽迁移期间也是允许的。
  这个标志在Redis Cluster部署中是相关的。
* **blocking：**该命令可能会阻塞请求的客户端。
* **denyoom：**如果服务器的内存使用过高（参见_maxmemory_配置指令），则拒绝执行该命令。
* **fast：**该命令在恒定时间或对数时间内运行。
  这个标志用于使用`LATENCY`命令监控延迟。
* **loading：**在数据库加载期间允许执行该命令。
* **movablekeys：**第一个键、最后一个键和步长值并不确定所有键的位置。
  在这种情况下，客户端需要使用`COMMAND GETKEYS`或键规范[td]。
  详见下文的更多细节。
* **no_auth：**执行该命令不需要身份验证。
* **no_async_loading：**异步加载期间禁止执行该命令（即当副本使用无磁盘`SWAPDB SYNC`时，并允许访问旧数据集）。
* **no_mandatory_keys：**该命令可以接受键名参数，但这些参数不是强制的。
* **no_multi：**在[事务](/topics/transactions)上下文中不允许执行该命令。
* **noscript：**该命令无法从[脚本](/topics/eval-intro)或[函数](/topics/functions-intro)调用。
* **pubsub：**该命令与[Redis Pub/Sub](/topics/pubsub)相关。
* **random：**该命令返回随机结果，这对于逐字脚本复制是个问题。
  截至Redis 7.0，该标志是一个[命令提示][tb]。
* **readonly：**该命令不修改数据。
* **sort_for_script：**当从脚本调用时，该命令的输出会进行排序。
* **skip_monitor：**该命令不会显示在`MONITOR`的输出中。
* **skip_slowlog：**该命令不会显示在`SLOWLOG`的输出中。
  截至Redis 7.0，该标志是一个[命令提示][tb]。
* **stale：**当副本具有陈旧数据时允许执行该命令。
* **write：**该命令可能会修改数据。

### 可移动的键

考虑`SORT`：

```
1) 1) "sort"
   2) (integer) -2
   3) 1) write
      2) denyoom
      3) movablekeys
   4) (integer) 1
   5) (integer) 1
   6) (integer) 1
   ...
```

一些Redis命令没有预定的键位置或难以找到。
对于这些命令，"movablekeys"标志表示使用"first key"、"last key"和"step"值是不足以找到所有键的。

下面是一些具有_movablekeys_标志的命令示例：

* `SORT`：可选的_STORE_、_BY_和_GET_修饰符后面跟着键的名称。
* `ZUNION`：_numkeys_参数指定键名参数的数量。
* `MIGRATE`：键出现 _KEYS_ 关键字，仅当第二个参数为空字符串时。

Redis集群客户端需要采用其他措施来定位这些命令的键，具体操作如下。

您可以使用`COMMAND GETKEYS`命令，并让您的Redis服务器报告给定命令调用的所有键。

截至 Redis 7.0 版本，客户端可以使用“键规范”来识别键名的位置。
只有解析键规范的客户端需要使用`COMMAND GETKEYS`命令，其中包括`SORT`和`MIGRATE`命令。

有关更多信息，请参阅[key specifications page][tr]。

## 第一个键

命令的第一个关键名称参数的位置。
对于大多数命令，第一个关键字的位置是1。
位置0始终是命令本身的名称。

## 最后一个按键

命令的最后一个关键名称参数的位置。
Redis命令通常接受一个、两个或多个键。

接受单个密钥的命令，即将 _第一个密钥_ 和 _最后一个密钥_ 均设置为 1。

接受两个键名参数的命令，如 `BRPOPLPUSH`、`SMOVE` 和 `RENAME`，将此值设置为其第二个键的位置。

多键命令接受任意数量的键，例如`MSET`，使用值-1。

## 步骤

步长（或增量）是指从第一个关键字到下一个关键字的位置之间的差距。

考虑以下两个例子：

```
1) 1) "mset"
   2) (integer) -3
   3) 1) write
      2) denyoom
   4) (integer) 1
   5) (integer) -1
   6) (integer) 2
   ...
```

```
1) 1) "mget"
   2) (integer) -2
   3) 1) readonly
      2) fast
   4) (integer) 1
   5) (integer) -1
   6) (integer) 1
   ...
```

计步器使我们能够找到键的位置。
例如`MSET`：其语法为`MSET _key1_ _val1_ [key2] [val2] [key3] [val3]...`，因此键位于每隔两个位置（步长值为_2_）。
与使用步长值为_1_的`MGET`不同。

## ACL分类

这是一个简单字符串的数组，这些字符串是命令所属的ACL类别。
请参考[访问控制列表][ta]页面获取更多信息。

## 命令提示

有关命令的有用信息。
供客户端/代理使用。

请查看有关更多信息的[命令提示][tb]页面。

## 关键规格

这是一个由命令关键规范组成的数组。
数组中的每个元素都是一个描述在命令参数中定位关键字的方法的映射。

请查看[关键规格页面][td]获取更多信息。

[td]: key specifications page's translated link

## 子命令

这是一个包含所有命令子命令的数组。如果有的话。
一些Redis命令有子命令（例如`CONFIG`的`REWRITE`子命令）。
数组中的每个元素表示一个子命令，并遵循与`COMMAND`返回值相同的规范。

[ta]: /topics/acl
[tb]: /topics/command-tips
[td]: /topics/key-specs
[tr]: /topics/key-specs

@examples

以下是`COMMAND`命令的输出结果，用于执行`GET`命令：

```
1)  1) "get"
    2) (integer) 2
    3) 1) readonly
       2) fast
    4) (integer) 1
    5) (integer) 1
    6) (integer) 1
    7) 1) @read
       2) @string
       3) @fast
    8) (empty array)
    9) 1) 1) "flags"
          2) 1) read
          3) "begin_search"
          4) 1) "type"
             2) "index"
             3) "spec"
             4) 1) "index"
                2) (integer) 1
          5) "find_keys"
          6) 1) "type"
             2) "range"
             3) "spec"
             4) 1) "lastkey"
                2) (integer) 0
                3) "keystep"
                4) (integer) 1
                5) "limit"
                6) (integer) 0
   10) (empty array)
...
```
