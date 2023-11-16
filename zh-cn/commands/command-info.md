返回关于多个 Redis 命令的详细信息的 @array-reply。

和`COMMAND`命令的结果格式相同，只是可以指定返回哪些命令。

如果您请求关于不存在命令的详细信息，它们的返回位置将为 nil。

@examples

```cli
COMMAND INFO get set eval
COMMAND INFO foo evalsha config bar
```
