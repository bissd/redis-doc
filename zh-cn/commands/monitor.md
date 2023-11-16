`MONITOR` 是一个调试命令，它会实时返回 Redis 服务器处理的每个命令。
它能够帮助了解数据库的运行情况。
这个命令可以通过 `redis-cli` 或者 `telnet` 使用。

查看由服务器处理的所有请求的能力在使用Redis作为数据库和分布式缓存系统时，对于发现应用程序中的错误非常有用。

```
$ redis-cli monitor
1339518083.107412 [0 127.0.0.1:60866] "keys" "*"
1339518087.877697 [0 127.0.0.1:60866] "dbsize"
1339518090.420270 [0 127.0.0.1:60866] "set" "x" "6"
1339518096.506257 [0 127.0.0.1:60866] "get" "x"
1339518099.363765 [0 127.0.0.1:60866] "eval" "return redis.call('set','x','7')" "0"
1339518100.363799 [0 lua] "set" "x" "7"
1339518100.544926 [0 127.0.0.1:60866] "del" "x"
```

请使用`SIGINT` (Ctrl-C) 停止通过 `redis-cli` 运行的 `MONITOR` 流。

```
$ telnet localhost 6379
Trying 127.0.0.1...
Connected to localhost.
Escape character is '^]'.
MONITOR
+OK
+1339518083.107412 [0 127.0.0.1:60866] "keys" "*"
+1339518087.877697 [0 127.0.0.1:60866] "dbsize"
+1339518090.420270 [0 127.0.0.1:60866] "set" "x" "6"
+1339518096.506257 [0 127.0.0.1:60866] "get" "x"
+1339518099.363765 [0 127.0.0.1:60866] "del" "x"
+1339518100.544926 [0 127.0.0.1:60866] "get" "x"
QUIT
+OK
Connection closed by foreign host.
```

使用`telnet`手动发送`QUIT`或`RESET`命令来停止运行中的`MONITOR`流。

## MONITOR不会记录的命令

由于安全问题，`MONITOR`的输出中不记录任何管理员命令，敏感数据在`AUTH`命令中被隐藏。

此外，命令`QUIT`也不会被记录。

## 运行MONITOR的成本

因为 `MONITOR` 流回 **所有** 命令，所以使用它是有代价的。
以下（完全不科学的）基准数据演示了运行 `MONITOR` 的代价。

没有运行`MONITOR`时的基准结果：

```
$ src/redis-benchmark -c 10 -n 100000 -q
PING_INLINE: 101936.80 requests per second
PING_BULK: 102880.66 requests per second
SET: 95419.85 requests per second
GET: 104275.29 requests per second
INCR: 93283.58 requests per second
```

使用`MONITOR`运行的基准测试结果(`redis-cli monitor > /dev/null`)：

```
$ src/redis-benchmark -c 10 -n 100000 -q
PING_INLINE: 58479.53 requests per second
PING_BULK: 59136.61 requests per second
SET: 41823.50 requests per second
GET: 45330.91 requests per second
INCR: 41771.09 requests per second
```

在这种特殊情况下，运行一个 `MONITOR` 客户端可以将吞吐量减少50%以上。
运行更多的 `MONITOR` 客户端会进一步降低吞吐量。

## 行为变更历史

*   `>= 6.0.0`：命令的输出中不包括`AUTH`。
*   `>= 6.2.0`：可以调用"`RESET`"退出监听模式。
*   `>= 6.2.4`：命令的输出中包括`AUTH`、`HELLO`、`EVAL`、`EVAL_RO`、`EVALSHA`和`EVALSHA_RO`。
