刷新Lua脚本缓存。

默认情况下，`SCRIPT FLUSH` 命令将同步地刷新缓存。
从 Redis 6.2 开始，将 `lazyfree-lazy-user-flush` 配置指令设置为 "yes" 会将默认的刷新模式更改为异步。

可以使用以下修饰符之一来显式地指定刷新模式：

* `ASYNC`：异步刷新缓存
* `!SYNC`：同步刷新缓存

关于`EVAL`脚本的更多信息，请参阅[EVAL脚本简介](/topics/eval-intro)。

## 行为变化历史

*   `>= 6.2.0`：默认的刷新行为现在可以通过**lazyfree-lazy-user-flush**配置指令进行配置。
