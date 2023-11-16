`CONFIG GET` 命令用于读取正在运行的 Redis 服务器的配置参数。
不是所有的配置参数在 Redis 2.4 中都受支持，而 Redis 2.6 可以使用此命令读取服务器的全部配置。

对称命令用于在运行时更改配置是 `CONFIG SET`。

`CONFIG GET` 命令可以接受多个参数，这些参数使用 glob 模式匹配。
与任何一个模式匹配的配置参数都会被报告为一个键值对列表。
示例：

```
redis> config get *max-*-entries* maxmemory
 1) "maxmemory"
 2) "0"
 3) "hash-max-listpack-entries"
 4) "512"
 5) "hash-max-ziplist-entries"
 6) "512"
 7) "set-max-intset-entries"
 8) "512"
 9) "zset-max-listpack-entries"
10) "128"
11) "zset-max-ziplist-entries"
12) "128"
```

您可以在打开的`redis-cli`提示符中输入`CONFIG GET *`来获取所有支持的配置参数列表。

所有支持的参数与[redis.conf][hgcarr22rc]文件中使用的等效配置参数具有相同的含义：

[hgcarr22rc]: http://github.com/redis/redis/raw/unstable/redis.conf

请注意，您应查看与您使用的版本相关的redis.conf文件，因为配置选项可能会在不同版本之间更改。上面的链接是到最新的开发版本。
