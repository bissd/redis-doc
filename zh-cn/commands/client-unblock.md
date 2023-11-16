这个命令可以解除不同连接中的一个在阻塞操作中被阻塞的客户端，比如`BRPOP`、`XREAD`或者`WAIT`。

默认情况下，如果命令的超时时间到达，客户端将被解除阻塞，但是如果传递了额外的（可选）参数，可以指定解除阻塞的行为，可以是**TIMEOUT**（默认值）或**ERROR**。如果指定了**ERROR**，行为是解除阻塞的同时将客户端返回错误，指示客户端是被强制解除阻塞的。具体而言，客户端将收到以下错误消息：

    -UNBLOCKED client unblocked via CLIENT UNBLOCK

注意：当然通常情况下不能保证错误文本保持不变，然而错误代码将保持“-UNBLOCKED”。

这个命令在我们用有限数量的连接监控多个键时特别有用。例如，我们可能想通过`XREAD`来监控多个流，但不想使用超过N个连接。然而，在某个点上，消费者进程会被通知有一个新的流键要监控。为了避免使用更多的连接，最好的做法是停止连接池中一条连接上的阻塞命令，添加新的键，然后再次发出阻塞命令。

为了获得这种行为，使用以下模式。该过程使用额外的*控制连接*来发送`CLIENT UNBLOCK`命令（如果需要）。同时，在对其他连接执行阻塞操作之前，该过程运行`CLIENT ID`以获取与该连接关联的ID。当应添加新键或停止监视键时，通过在控制连接中发送`CLIENT UNBLOCK`来中止相关的连接阻塞命令。阻塞命令将返回，最后可以重新发出。

该示例展示了Redis流应用的情景，然而这种模式是通用的，可以应用于其他情况。

@examples

```
Connection A (blocking connection):
> CLIENT ID
2934
> BRPOP key1 key2 key3 0
(client is blocked)

... Now we want to add a new key ...

Connection B (control connection):
> CLIENT UNBLOCK 2934
1

Connection A (blocking connection):
... BRPOP reply with timeout ...
NULL
> BRPOP key1 key2 key3 key4 0
(client is blocked again)
```
