---
title: "Redis CLI"
linkTitle: "CLI"
weight: 1
description: >
    redis-cli是Redis的命令行界面概述
aliases:
    - /docs/manual/cli
    - /docs/management/cli
    - /docs/ui/cli
---

在交互模式下，`redis-cli`具有基本的行编辑功能，以提供一个熟悉的键入体验。

为了以特殊模式启动程序，您可以使用几个选项，包括：

* 模拟复制品并打印从主服务器接收到的复制流。
* 检查 Redis 服务器的延迟并显示统计信息。
* 请求延迟样本和频率的 ASCII 艺术谱图。

本主题涵盖了`redis-cli`的不同方面，从最简单的开始，到更高级的功能结束。

## 命令行使用方法

为了在终端上运行Redis命令并返回标准输出，请将要执行的命令作为`redis-cli`的单独参数进行包含：

    $ redis-cli INCR mycounter
    (integer) 7

该命令的回复是"7"。由于Redis的回复是有类型的（字符串、数组、整数、空值、错误等），在括号中可以看到回复的类型。当`redis-cli`的输出必须用作另一个命令的输入或重定向到文件时，这种额外的信息可能不是理想的。

`redis-cli` 仅在检测到标准输出是 tty 或终端时，才显示附加信息以供人类阅读。对于所有其他输出，它将自动启用*原始输出模式*，如以下示例所示：

    $ redis-cli INCR mycounter > /tmp/output.txt
    $ cat /tmp/output.txt
    8

请注意，输出中省略了`(integer)`，因为 `redis-cli` 检测到输出不再写入终端。您可以使用 `--raw` 选项在终端上强制原始输出：

    $ redis-cli --raw INCR mycounter
    9

您可以在写入文件或通过使用`--no-raw`在管道中传递给其他命令时，强制生成可读的输出。

## 字符串引用和转义

当 `redis-cli` 解析命令时，空白字符会自动分隔参数。
在交互模式下，换行符会发送命令进行解析和执行。
要输入包含空白字符或不可打印字符的字符串值，可以使用带引号和转义的字符串。

引用字符串值需要使用双引号（"）或单引号（'）括起来。
转义序列用于在字符和字符串字面值中插入不可打印字符。

一个转义序列包含一个反斜杠（`\`）符号，后面跟着一个转义序列字符。

双引号的字符串支持以下转义序列：

- `\"` - 双引号
- `\n` - 换行
- `\r` - 回车
- `\t` - 横向制表符
- `\b` - 退格
- `\a` - 警报
- `\\` - 反斜杠
- `\xhh` - 由十六进制数字 (_hh_) 表示的任意ASCII字符

单引号假设字符串是字面值，并且只允许以下转义序列：
* `\'` - 单引号
* `\\` - 反斜杠

例如，要在两行上返回“Hello World”：

```
127.0.0.1:6379> SET mykey "Hello\nWorld"
OK
127.0.0.1:6379> GET mykey
Hello
World
```

在输入包含单引号或双引号的字符串时，例如在密码中，必须对字符串进行转义，例如:

```
127.0.0.1:6379> AUTH some_admin_user ">^8T>6Na{u|jp>+v\"55\@_;OU(OR]7mbAYGqsfyu48(j'%hQH7;v*f1H${*gD(Se'"
 ```

## 主机、端口、密码和数据库

默认情况下，`redis-cli` 会连接到地址为 127.0.0.1 端口为 6379 的服务器。
您可以使用多个命令行选项来更改端口。要指定不同的主机名或 IP 地址，请使用 `-h` 选项。要设置不同的端口，请使用 `-p` 选项。

    $ redis-cli -h redis15.localnet.org -p 6390 PING
    PONG

如果您的实例受密码保护，请使用“-a <password>”选项进行身份验证，而无需显式使用“AUTH”命令：

    $ redis-cli -a myUnguessablePazzzzzword123 PING
    PONG

**注意：** 出于安全原因，通过`REDISCLI_AUTH`环境变量自动向`redis-cli`提供密码。

最后，通过使用"-n<dbnum>"选项，可以发送一个操作非默认数据库编号零的命令。

    $ redis-cli FLUSHALL
    OK
    $ redis-cli -n 1 INCR a
    (integer) 1
    $ redis-cli -n 1 INCR a
    (integer) 2
    $ redis-cli -n 2 INCR a
    (integer) 1

部分或全部的信息也可以通过使用`-u <uri>`选项和URI模式`redis://user:password@host:port/dbnum`提供：

    $ redis-cli -u redis://LJenkins:p%40ssw0rd@redis-16379.hosted.com:16379/0 PING
    PONG

## SSL/TLS
## SSL/TLS

默认情况下，`redis-cli` 使用普通的 TCP 连接连接到 Redis。
您可以使用 `--tls` 选项启用 SSL/TLS，同时使用 `--cacert` 或 `--cacertdir` 配置受信任的根证书捆绑包或目录。

如果目标服务器需要使用客户端证书进行身份验证，则可以使用`--cert`和`--key`指定证书和相应的私钥。

## 从其他程序获取输入

有两种方法可以使用`redis-cli`来接收其他命令通过标准输入传递的输入。一种方法是使用目标负载作为从*stdin*中的最后一个参数。例如，为了将Redis键`net_services`设置为来自本地文件系统的文件`/etc/services`的内容，请使用`-x`选项：

    $ redis-cli -x SET net_services < /etc/services
    OK
    $ redis-cli GETRANGE net_services 0 50
    "#\n# Network services, Internet style\n#\n# Note that "

在上面会话的第一行中，使用`redis-cli`执行了`-x`选项，并将一个文件重定向到CLI的标准输入作为
满足`SET net_services`命令短语的值。这对于脚本编写很有用。

以文本文件形式将一系列命令写入`redis-cli`是一种不同的方法：

    $ cat /tmp/commands.txt
    SET item:3374 100
    INCR item:3374
    APPEND item:3374 xxx
    GET item:3374
    $ cat /tmp/commands.txt | redis-cli
    OK
    (integer) 101
    (integer) 6
    "101xxx"

命令.txt中的所有命令将按顺序由redis-cli执行，就像用户在交互模式下输入一样。如果需要，可以在文件中引用字符串，以便可以包含带有空格、换行符或其他特殊字符的单个参数：

    $ cat /tmp/commands.txt
    SET arg_example "This is a single argument"
    STRLEN arg_example
    $ cat /tmp/commands.txt | redis-cli
    OK
    (integer) 25

## 持续运行相同命令

可以执行指定次数的单个命令，并在执行之间选择暂停时间。这在不同的情境下非常有用-例如，当我们想要连续监视一些关键内容或`INFO`字段输出时，或者当我们想要模拟一些重复写入事件，比如每5秒钟向列表中推送一个新项时。

这个功能由两个选项控制：`-r <count>`和`-i <delay>`。
`-r`选项指定运行命令的次数，`-i`选项设置不同命令调用之间的延迟，以秒为单位（可以指定值如0.1表示100毫秒）。

默认情况下，时间间隔（或延迟）设置为0，因此命令会立即执行：

    $ redis-cli -r 5 INCR counter_value
    (integer) 1
    (integer) 2
    (integer) 3
    (integer) 4
    (integer) 5

要无限运行相同的命令，请将计数值设置为`-1`。
要随时间监测 RSS 内存大小，可以使用以下命令：

    $ redis-cli -r -1 -i 1 INFO | grep rss_human
    used_memory_rss_human:2.71M
    used_memory_rss_human:2.73M
    used_memory_rss_human:2.73M
    used_memory_rss_human:2.73M
    ... a new line will be printed each second ...

## 使用`redis-cli`进行大规模数据插入

`redis-cli` 是一个值得探讨的主题，有关使用 `redis-cli` 进行批量插入的信息在单独的页面上进行了介绍。请参考我们的[批量插入指南](/topics/mass-insert)。

## CSV 输出

Redis-cli具有CSV（逗号分隔值）输出功能，可以将数据从Redis导出到外部程序。

    $ redis-cli LPUSH mylist a b c d
    (integer) 4
    $ redis-cli --csv LRANGE mylist 0 -1
    "d","c","b","a"

请注意，“--csv”标志只适用于单个命令，而不适用于整个数据库的导出。

## 运行 Lua 脚本

`redis-cli`对Lua脚本的调试功能有广泛的支持，该功能在Redis 3.2及更高版本中可用。有关该功能，请参阅[Redis Lua调试器文档](/topics/ldb)。

即使不使用调试器，也可以使用`redis-cli`来从文件中运行脚本作为参数：

    $ cat /tmp/script.lua
    return redis.call('SET',KEYS[1],ARGV[1])
    $ redis-cli --eval /tmp/script.lua location:hastings:temp , 23
    OK

Redis的`EVAL`命令将脚本使用的键列表和其他非键参数作为不同的数组。在调用`EVAL`时，您需要提供键的数量作为一个数字。

在调用`redis-cli`时使用上面的`--eval`选项时，无需显式指定键的数量。相反，它使用以逗号分隔键和参数的约定。这就是为什么在上面的调用中你会看到`location:hastings:temp , 23`作为参数的原因。

所以 `location:hastings:temp` 将填充 `KEYS` 数组，`23` 填充 `ARGV` 数组。

`--eval` 选项在编写简单脚本时很有用。对于更复杂的工作，建议使用 Lua 调试器。可以混合使用这两种方法，因为调试器也可以执行来自外部文件的脚本。

互动模式
===


我们已经探索了如何将Redis CLI用作命令行程序。
这对于脚本和某些类型的测试非常有用，但是大多数人在`redis-cli`中使用它的交互式模式时会花费大多数时间。

在交互模式下，用户在提示符下输入Redis命令。命令被发送到服务器进行处理，答复被解析并呈现为更简单的形式用于阅读。

在交互模式下运行 `redis-cli` 不需要任何特殊操作 - 只需不带任何参数执行即可。

    $ redis-cli
    127.0.0.1:6379> PING
    PONG

字符串 "127.0.0.1:6379>" 是提示符。它显示了已连接的Redis服务器实例的主机名和端口。

prompt 提示符将会在与服务器连接发生变化时或者在操作非 0 号数据库时更新：

    127.0.0.1:6379> SELECT 2
    OK
    127.0.0.1:6379[2]> DBSIZE
    (integer) 1
    127.0.0.1:6379[2]> SELECT 0
    OK
    127.0.0.1:6379> DBSIZE
    (integer) 503

## 处理连接和重新连接

在交互模式下使用`CONNECT`命令可以连接到不同的实例，通过指定我们想要连接的*主机名*和*端口*。

    127.0.0.1:6379> CONNECT metal 6379
    metal:6379> PING
    PONG

正如您所见，在连接到不同的服务器实例时，提示会相应地更改。
如果尝试连接到一个无法访问的实例，`redis-cli` 将进入断开连接模式，并尝试在每个新命令中重新连接：

    127.0.0.1:6379> CONNECT 127.0.0.1 9999
    Could not connect to Redis at 127.0.0.1:9999: Connection refused
    not connected> PING
    Could not connect to Redis at 127.0.0.1:9999: Connection refused
    not connected> PING
    Could not connect to Redis at 127.0.0.1:9999: Connection refused

一般在检测到断开连接后，`redis-cli`总是尝试透明地重新连接；如果尝试失败，则显示错误并进入断开连接状态。以下是一个断开连接和重新连接的示例：

    127.0.0.1:6379> INFO SERVER
    Could not connect to Redis at 127.0.0.1:6379: Connection refused
    not connected> PING
    PONG
    127.0.0.1:6379> 
    (now we are connected again)

当重新连接时，`redis-cli`会自动重新选择最后一个选择的数据库编号。但是，关于连接的其他状态都会丢失，比如在MULTI/EXEC事务中的状态：

    $ redis-cli
    127.0.0.1:6379> MULTI
    OK
    127.0.0.1:6379> PING
    QUEUED

    ( here the server is manually restarted )

    127.0.0.1:6379> EXEC
    (error) ERR EXEC without MULTI

在测试时，通常使用交互模式的`redis-cli`不会出现此问题，但您应该知道这个限制。

## 编辑、历史记录、完成和提示

因为 `redis-cli` 使用了 [linenoise line editing library](http://github.com/antirez/linenoise)，所以它始终具备行编辑功能，而不依赖于 `libreadline` 或其他可选库。

命令执行历史记录可以通过按上下箭头键访问，以避免重新输入命令。
历史记录会在CLI重启时保留，在用户主目录中的名为`.rediscli_history`的文件中，由`HOME`环境变量指定。
可以通过设置`REDISCLI_HISTFILE`环境变量来使用不同的历史记录文件名，并通过将其设置为`/dev/null`来禁用它。

`redis-cli`还可以通过按下TAB键来执行命令名称自动完成，如下面的示例所示：

    127.0.0.1:6379> Z<TAB>
    127.0.0.1:6379> ZADD<TAB>
    127.0.0.1:6379> ZCARD<TAB>

一旦在提示符处输入了Redis命令名称，`redis-cli`将显示语法提示。与命令历史记录一样，可以通过`redis-cli`首选项启用或禁用此行为。

## 首选项

有两种方式可以自定义`redis-cli`的行为。CLI在启动时会加载位于主目录下的`.redisclirc`文件。您可以通过设置`REDISCLI_RCFILE`环境变量为其他路径来覆盖文件的默认位置。也可以在CLI会话期间设置偏好设置，这种情况下偏好设置将只在会话持续期间生效。

为了设置首选项，请使用特殊命令 `:set`。可以设置以下首选项，
可以通过在 CLI 中键入命令或将其添加到 `.redisclirc` 文件中进行设置：

* `:set hints` - 启用语法提示
* `:set nohints` - 禁用语法提示

以相同的命令运行N次

在交互模式下，通过在命令名称前面加上一个数字，可以多次运行相同的命令。

    127.0.0.1:6379> 5 INCR mycounter
    (integer) 1
    (integer) 2
    (integer) 3
    (integer) 4
    (integer) 5

## 展示有关Redis命令的帮助信息

`redis-cli` 提供了对大多数 Redis [命令](/commands) 的在线帮助，使用 `HELP` 命令。该命令有两种形式可用：

* `HELP @<category>`显示关于给定类别的所有命令。类别包括：
    - `@generic`
    - `@string`
    - `@list`
    - `@set`
    - `@sorted_set`
    - `@hash`
    - `@pubsub`
    - `@transactions`
    - `@connection`
    - `@server`
    - `@scripting`
    - `@hyperloglog`
    - `@cluster`
    - `@geo`
    - `@stream`
* `HELP <commandname>`显示给定命令的具体帮助信息。

为了显示`PFADD`命令的帮助信息，可以使用以下示例：

    127.0.0.1:6379> HELP PFADD

    PFADD key element [element ...]
    summary: Adds the specified elements to the specified HyperLogLog.
    since: 2.8.9

请注意`HELP`也支持TAB自动补全。

## 清除终端屏幕

在交互模式下使用`CLEAR`命令可以清除终端屏幕。

特殊操作模式
===

到目前为止，我们看到了 `redis-cli` 的两种主要模式。

* Redis命令的命令行执行。
* 可交互式的“REPL”使用。

命令行界面（CLI）执行与Redis相关的其他辅助任务，这些任务在接下来的章节中进行解释：

* 用于显示关于Redis服务器的持续统计信息的监控工具。
* 扫描Redis数据库以查找非常大的键。
* 具有模式匹配的键空间扫描器。
* 作为[Pub/Sub](/topics/pubsub)客户端订阅频道。
* 监控执行到Redis实例的命令。
* 以不同方式检查Redis服务器的[延迟](/topics/latency)。
* 检查本地计算机的调度器延迟。
* 将远程Redis服务器的RDB备份传输到本地。
* 充当Redis副本，以显示副本接收到的内容。
* 模拟[LRU](/topics/lru-cache)工作负载，以显示有关键击中的统计信息。
* Lua调试器的客户端。

## 持续统计模式

连续统计模式可能是`redis-cli`中较少人知道但非常有用的功能之一，可用于实时监控Redis实例。要启用此模式，使用`--stat`选项。
在此模式下，输出非常清晰地反映了CLI的行为：

    $ redis-cli --stat
    ------- data ------ --------------------- load -------------------- - child -
    keys       mem      clients blocked requests            connections
    506        1015.00K 1       0       24 (+0)             7
    506        1015.00K 1       0       25 (+1)             7
    506        3.40M    51      0       60461 (+60436)      57
    506        3.40M    51      0       146425 (+85964)     107
    507        3.40M    51      0       233844 (+87419)     157
    507        3.40M    51      0       321715 (+87871)     207
    508        3.40M    51      0       408642 (+86927)     257
    508        3.40M    51      0       497038 (+88396)     257

在这个模式下，每秒打印一行带有有用信息和旧数据点之间请求值的差异。可以通过这个辅助`redis-cli`工具轻松理解关于连接的 Redis 数据库的内存使用情况、客户端连接计数和其他各种统计信息。

在这种情况下，`-i <interval>`选项作为修改器，用于更改发出新行的频率。默认值为一秒钟。

## 扫描大键

在这种特殊模式下，`redis-cli`作为一个键空间分析器工作。它扫描数据集以查找大键，并提供关于数据集所包含的数据类型的信息。可以使用`--bigkeys`选项启用此模式，并生成详细输出：

    $ redis-cli --bigkeys

    # Scanning the entire keyspace to find biggest keys as well as
    # average sizes per key type.  You can use -i 0.01 to sleep 0.01 sec
    # per SCAN command (not usually needed).

    [00.00%] Biggest string found so far 'key-419' with 3 bytes
    [05.14%] Biggest list   found so far 'mylist' with 100004 items
    [35.77%] Biggest string found so far 'counter:__rand_int__' with 6 bytes
    [73.91%] Biggest hash   found so far 'myobject' with 3 fields

    -------- summary -------

    Sampled 506 keys in the keyspace!
    Total key length in bytes is 3452 (avg len 6.82)

    Biggest string found 'counter:__rand_int__' has 6 bytes
    Biggest   list found 'mylist' has 100004 items
    Biggest   hash found 'myobject' has 3 fields

    504 strings with 1403 bytes (99.60% of keys, avg size 2.78)
    1 lists with 100004 items (00.20% of keys, avg size 100004.00)
    0 sets with 0 members (00.00% of keys, avg size 0.00)
    1 hashs with 3 fields (00.20% of keys, avg size 3.00)
    0 zsets with 0 members (00.00% of keys, avg size 0.00)

在输出的第一部分中，报告了每个新的键大于前一个遇到的相同类型键的情况。摘要部分提供了关于Redis实例中数据的一般统计信息。

程序使用`SCAN`命令，因此可以在繁忙的服务器上执行而不影响操作，但是可以使用`-i`选项来调节每个`SCAN`命令的扫描进程以秒为单位的指定部分。

例如，`-i 0.01` 将显著降低程序的执行速度，但也将极大地减少服务器的负载，几乎可以忽略不计。

请注意，摘要也以更简洁的形式报告了每次发现的最大关键字。初始输出仅为了在运行非常大的数据集时提供一些有趣的信息。

## 获取键的列表

也可以以一种不阻塞Redis服务器的方式扫描键空间（使用`KEYS *`命令会导致阻塞），并打印出所有键名，或者根据特定模式筛选键名。这种模式和`--bigkeys`选项一样，使用`SCAN`命令，所以如果数据集发生变化，可能会多次报告键，但如果该键从迭代开始就存在，则不会丢失任何键。由于使用这个选项的命令是`--scan`，因此它被称为“扫描”选项。

    $ redis-cli --scan | head -10
    key-419
    key-71
    key-236
    key-50
    key-38
    key-458
    key-453
    key-499
    key-446
    key-371


注意，在输出中只打印前十行时使用了 `head -10`。

扫描能够利用`SCAN`命令的底层模式匹配能力，使用`--pattern`选项。

    $ redis-cli --scan --pattern '*-11*'
    key-114
    key-117
    key-118
    key-113
    key-115
    key-112
    key-119
    key-11
    key-111
    key-110
    key-116

通过将输出通过 "wc" 命令传递，可以按键名计算特定类型的对象数量：

    $ redis-cli --scan --pattern 'user:*' | wc -l
    3829433

您可以使用`-i 0.01`在调用`SCAN`命令之间添加延迟。
这将使命令变慢，但会显著减轻服务器负载。

## Pub/sub 模式

CLI能够使用`PUBLISH`命令在Redis Pub/Sub频道中发布消息。订阅频道以接收消息的方式有所不同 - 终端会被阻塞并等待消息，因此这在`redis-cli`中是作为一种特殊模式实现的。与其他特殊模式不同的是，不需要使用特殊选项来启用此模式，而只需使用`SUBSCRIBE`或`PSUBSCRIBE`命令即可，这些命令可以在交互模式或命令模式下使用。

    $ redis-cli PSUBSCRIBE '*'
    Reading messages... (press Ctrl-C to quit)
    1) "PSUBSCRIBE"
    2) "*"
    3) (integer) 1

*阅读消息*消息显示我们进入了发布/订阅模式。
当另一个客户端在某个频道上发布一些消息时，比如使用命令 `redis-cli PUBLISH mychannel mymessage`，处于发布/订阅模式的CLI将会显示类似下面的内容：

    1) "pmessage"
    2) "*"
    3) "mychannel"
    4) "mymessage"

这对于调试Pub/Sub问题非常有用。
要退出Pub/Sub模式，请使用`CTRL-C`命令。

## 监控在Redis中执行的命令

与发布/订阅模式类似，一旦使用`MONITOR`命令，将自动进入监控模式。所有接收到的命令都会打印到标准输出。

    $ redis-cli MONITOR
    OK
    1460100081.165665 [0 127.0.0.1:51706] "set" "shipment:8000736522714:status" "sorting"
    1460100083.053365 [0 127.0.0.1:51707] "get" "shipment:8000736522714:status"

注意可以使用管道将输出进行传输，这样您就可以使用诸如"grep"的工具来监视特定模式。

## 监控Redis实例的延迟

在需要非常重视延迟的情况下，Redis经常被使用。延迟涉及到应用程序中的多个组成部分，从客户端库到网络堆栈，再到Redis实例本身。

`redis-cli`具有多种工具用于研究Redis实例的延迟，并了解延迟的最大值、平均值和分布情况。

基本的延迟检测工具是`--latency`选项。使用此选项，CLI运行一个循环，将`PING`命令发送到Redis实例，并测量接收回复的时间。这会每秒发生100次，并且统计数据实时在控制台更新。

    $ redis-cli --latency
    min: 0, max: 1, avg: 0.19 (427 samples)

统计数据以毫秒为单位。通常，由于运行`redis-cli`的系统的内核调度程序引起的延迟，非常快的实例的平均延迟往往会被高估一点，所以上面的0.19的平均延迟可能很容易变成0.01或更低。然而，这通常不是一个大问题，因为大多数开发人员对几毫秒或更长时间的事件感兴趣。

有时候研究最大延迟和平均延迟的演变过程是很有用的。`--latency-history`选项用于这个目的：它的工作方式与`--latency`完全相同，但是每隔15秒（默认值）就会从头开始一个新的取样会话：

    $ redis-cli --latency-history
    min: 0, max: 1, avg: 0.14 (1314 samples) -- 15.01 seconds range
    min: 0, max: 1, avg: 0.18 (1299 samples) -- 15.00 seconds range
    min: 0, max: 1, avg: 0.20 (113 samples)^C

使用`-i <间隔>`选项可以更改采样会话的长度。

最先进的延迟研究工具，但对于非经验用户来说，也是最难解释的，是使用彩色终端显示延迟频谱的能力。您将看到一个彩色输出，指示不同样本的百分比，并使用不同的ASCII字符指示不同的延迟数据。使用`--latency-dist`选项启用此模式：

    $ redis-cli --latency-dist
    (output not displayed, requires a color terminal, try it!)

还有另一个非常不寻常的延迟工具在`redis-cli`内部实现。
它不会检查Redis实例的延迟，而是检查运行`redis-cli`的计算机的延迟。
这种延迟是内核调度器的固有延迟，虚拟化实例中的超级管理程序等造成的。

Redis将其称为*固有延迟*，因为它对程序员来说大部分是不透明的。
如果Redis实例的延迟很高，无论有哪些明显的原因可能是源头，都值得检查在运行Redis服务器的系统中，通过在特殊模式下直接运行`redis-cli`来确定系统能够做到最好的情况。

通过测量内在延迟，您可以知道这是基准，
Redis无法胜过您的系统。为了以此模式运行CLI，
请使用`--intrinsic-latency <test-time>`。注意，测试时间以秒为单位，指示测试应该运行多长时间。

    $ ./redis-cli --intrinsic-latency 5
    Max latency so far: 1 microseconds.
    Max latency so far: 7 microseconds.
    Max latency so far: 9 microseconds.
    Max latency so far: 11 microseconds.
    Max latency so far: 13 microseconds.
    Max latency so far: 15 microseconds.
    Max latency so far: 34 microseconds.
    Max latency so far: 82 microseconds.
    Max latency so far: 586 microseconds.
    Max latency so far: 739 microseconds.

    65433042 total runs (avg latency: 0.0764 microseconds / 764.14 nanoseconds per run).
    Worst run took 9671x longer than the average latency.

重要提示：此命令必须在运行Redis服务器实例的计算机上执行，而不是在其他主机上执行。它不会连接到Redis实例并在本地执行测试。

在上述情况下，系统的最坏情况延迟不能超过739微秒，因此可以预期某些查询偶尔会运行少于1毫秒。

## RDB文件的远程备份

在Redis复制的第一次同步过程中，主节点和从节点会通过RDB文件的形式交换整个数据集。 `redis-cli`利用这个功能提供了远程备份功能，允许将RDB文件从任何Redis实例传输到运行`redis-cli`的本地计算机。要使用此模式，请使用`--rdb <dest-filename>`选项调用CLI：

    $ redis-cli --rdb /tmp/dump.rdb
    SYNC sent to master, writing 13256 bytes to '/tmp/dump.rdb'
    Transfer finished with success.

这是一种简单但有效的方法，可以确保您的Redis实例存在RDB备份。在脚本或cron作业中使用此选项时，请确保检查命令的返回值。如果返回值为非零，则发生了错误，如以下示例所示：

    $ redis-cli --rdb /tmp/dump.rdb
    SYNC with master failed: -ERR Can't SYNC while not connected with my master
    $ echo $?
    1

## 复制模式

CLI的复制模式是一项高级功能，对于Redis开发人员和调试操作非常有用。
它允许检查主节点发送给复制节点的内容，以便将写操作传播到复制节点。选项名为`--replica`。以下是一个工作示例：

    $ redis-cli --replica
    SYNC with master, discarding 13256 bytes of bulk transfer...
    SYNC done. Logging commands from master.
    "PING"
    "SELECT","0"
    "SET","last_name","Enigk"
    "PING"
    "INCR","mycounter"

命令开始时丢弃第一次同步的 RDB 文件，
然后将每个收到的命令记录在 CSV 格式中。

如果您认为其中一些命令在副本中没有正确复制，那么这是检查发生情况的好方法，同时也是改进错误报告的有用信息。

## 执行LRU模拟

通常，Redis被用作使用LRU回收作为缓存。取决于键的数量和为缓存分配的内存量（通过'maxmemory'指令指定），缓存命中和未命中的数量将会改变。有时，模拟命中率非常有用，以正确配置您的缓存。

`redis-cli`有一个特殊的模式，在这个模式下它会模拟GET和SET操作，使用80-20%的幂率分布作为请求模式。
这意味着20%的键会被80%的时间请求，这在缓存场景中是一个常见的分布方式。

理论上，根据请求分布和Redis内存开销，可以通过数学公式计算命中率。然而，Redis可以配置不同的LRU设置（样本数量），并且Redis中用于近似LRU算法的实现在不同版本之间有很大变化。同样，每个键的内存量也会因版本而异。这就是为什么构建了这个工具：它的主要动机是测试Redis的LRU实现的质量，但现在也适用于测试给定版本在部署中的行为方式。

使用此模式时，请指定测试中的键数，并将合理的 `maxmemory` 设置为首次尝试。

## 重要提示：在Redis配置中设置`maxmemory`参数是至关重要的，因为如果对最大内存使用量没有限制，那么全部的键都将存储在内存中，最终hit率将达到100%。如果指定的键过多，最终会使用完所有计算机的内存。此外，还需要配置一个适当的*maxmemory策略*；大多数情况下选择`allkeys-lru`。

在以下示例中，配置了100MB的内存限制和使用1000万个键的LRU模拟。

警告：该测试使用流水线技术，会对服务器进行压力测试，请不要在生产环境实例中使用。

    $ ./redis-cli --lru-test 10000000
    156000 Gets/sec | Hits: 4552 (2.92%) | Misses: 151448 (97.08%)
    153750 Gets/sec | Hits: 12906 (8.39%) | Misses: 140844 (91.61%)
    159250 Gets/sec | Hits: 21811 (13.70%) | Misses: 137439 (86.30%)
    151000 Gets/sec | Hits: 27615 (18.29%) | Misses: 123385 (81.71%)
    145000 Gets/sec | Hits: 32791 (22.61%) | Misses: 112209 (77.39%)
    157750 Gets/sec | Hits: 42178 (26.74%) | Misses: 115572 (73.26%)
    154500 Gets/sec | Hits: 47418 (30.69%) | Misses: 107082 (69.31%)
    151250 Gets/sec | Hits: 51636 (34.14%) | Misses: 99614 (65.86%)

程序每秒显示统计数据。在最开始的几秒钟内，缓存开始填充。后来的缺失率稳定在可以预期的实际数值上。

    120750 Gets/sec | Hits: 48774 (40.39%) | Misses: 71976 (59.61%)
    122500 Gets/sec | Hits: 49052 (40.04%) | Misses: 73448 (59.96%)
    127000 Gets/sec | Hits: 50870 (40.06%) | Misses: 76130 (59.94%)
    124250 Gets/sec | Hits: 50147 (40.36%) | Misses: 74103 (59.64%)

对于某些用例，59%的错误率可能是无法接受的，因此100MB的内存不足够。观察使用半GB内存的一个示例。经过几分钟后，输出数据稳定为以下数值：

    140000 Gets/sec | Hits: 135376 (96.70%) | Misses: 4624 (3.30%)
    141250 Gets/sec | Hits: 136523 (96.65%) | Misses: 4727 (3.35%)
    140250 Gets/sec | Hits: 135457 (96.58%) | Misses: 4793 (3.42%)
    140500 Gets/sec | Hits: 135947 (96.76%) | Misses: 4553 (3.24%)

用500MB的空间足够存储关键数量（1000万个）和分布（80-20样式）。
