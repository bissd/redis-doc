这条命令将完全重置连接的服务器端上下文，模拟断开连接并重新连接的效果。

当从常规客户端连接调用该命令时，它会执行以下操作：

* 如果存在当前的`MULTI`事务块，则丢弃它。
* 解除连接所监视的所有键的`WATCH`。
* 禁用`CLIENT TRACKING`，如果使用中。
* 设置连接为`READWRITE`模式。
* 取消连接的`ASKING`模式，如果之前设置过。
* 设置`CLIENT REPLY`为`ON`。
* 将协议版本设置为RESP2。
* `SELECT`数据库0。
* 在适用的情况下退出`MONITOR`模式。
* 在适当的情况下，取消Pub/Sub的订阅状态（`SUBSCRIBE`和`PSUBSCRIBE`）。
* 取消连接的身份验证，需要调用`AUTH`进行重新认证，当启用身份验证时。
* 关闭`NO-EVICT`模式。
* 关闭`NO-TOUCH`模式。
