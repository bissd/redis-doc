`CLIENT KILL`命令关闭给定的客户端连接。该命令支持两种格式，旧格式：

    CLIENT KILL addr:port

`ip:port` 应该与 `CLIENT LIST` 命令返回的一行匹配(`addr` 字段)。

新格式：

    CLIENT KILL <filter> <value> ... ... <filter> <value>

用新的形式可以根据不同属性杀死客户端，而不仅仅是根据地址。以下过滤器可用：

* `CLIENT KILL ADDR ip:port`。与旧的三参数行为完全相同。
* `CLIENT KILL LADDR ip:port`。关闭所有连接到指定本地（绑定）地址的客户端。
* `CLIENT KILL ID client-id`。通过其唯一的`ID`字段关闭客户端。可以使用`CLIENT LIST`命令获取客户端的`ID`。
* `CLIENT KILL TYPE type`，其中*type*是`normal`、`master`、`replica`和`pubsub`中的一个。关闭指定类别中的所有客户端的连接。请注意，被阻塞在`MONITOR`命令中的客户端被视为属于`normal`类别。
* `CLIENT KILL USER username`。关闭所有使用指定的[ACL](/topics/acl)用户名进行身份验证的连接，但如果用户名与现有的ACL用户不匹配，则返回错误。
* `CLIENT KILL SKIPME yes/no`。默认情况下，此选项设置为`yes`，即调用该命令的客户端不会被关闭，但将此选项设置为`no`将导致调用该命令的客户端也被关闭。

可以同时提供多个过滤器。命令将通过逻辑与处理多个过滤器。例如：

    CLIENT KILL addr 127.0.0.1:12345 type pubsub

以下文本是有效的，并且将只关闭具有指定地址的发布订阅客户端。当前情况下，包含多个过滤器的这种格式很少有用。

使用新的表单时，命令不再返回`OK`或错误，而是返回被终止的客户端数量，可能为零。

## 客户端终止和Redis Sentinel

最新版本的Redis Sentinel（Redis 2.8.12或更高版本）使用CLIENT KILL来杀死客户端，当实例重新配置时，以强制客户端再次与一个Sentinel进行握手并更新其配置。

## 笔记

由于Redis的单线程特性，当客户端执行命令时无法关闭客户端连接。从客户端的角度来看，在命令执行过程中连接永远不会被关闭。然而，只有在发送下一条命令时（并导致网络错误）客户端才会注意到连接已关闭。
