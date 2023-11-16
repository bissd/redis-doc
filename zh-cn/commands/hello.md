切换到一个不同的协议，可选择身份验证和设置连接名称，或提供上下文客户端报告。

Redis版本6及以上支持两个协议：旧协议RESP2和在Redis 6中引入的新协议RESP3。 RESP3具有某些优势，因为在该模式下，Redis能够提供更多语义回复：例如，`HGETALL`将返回一个*映射类型*，因此客户端库实现不再需要提前将数组转换为哈希再返回给调用方。有关RESP3的完整覆盖，请查看[RESP3规范](https://github.com/redis/redis-specifications/blob/master/protocol/RESP3.md)。

在Redis 6中，连接以RESP2模式开始，因此实现RESP2的客户端无需进行更新或更改。暂时没有放弃对RESP2的支持的计划，尽管将来的版本可能默认使用RESP3。

`HELLO`始终返回当前服务器和连接属性的列表，例如版本、加载的模块、客户端ID、复制角色等等。在Redis 6.2及其对RESP2协议的默认用法中，在不带任何参数的情况下调用时，回复的格式如下：

    > HELLO
     1) "server"
     2) "redis"
     3) "version"
     4) "255.255.255"
     5) "proto"
     6) (integer) 2
     7) "id"
     8) (integer) 5
     9) "mode"
    10) "standalone"
    11) "role"
    12) "master"
    13) "modules"
    14) (empty array)

客户端想要使用RESP3模式进行握手，需要调用`HELLO`命令并将值"3"作为`protover`参数指定，如下所示：

    > HELLO 3
    1# "server" => "redis"
    2# "version" => "6.0.0"
    3# "proto" => (integer) 3
    4# "id" => (integer) 10
    5# "mode" => "standalone"
    6# "role" => "master"
    7# "modules" => (empty array)

因为 `HELLO` 命令会回复有用的信息，并且考虑到 `protover` 是可选项或者可以设置为 "2"，客户端库的作者可以考虑在建立连接时使用这个命令，而不是常规的 `PING` 命令。

当使用可选的`protover`参数调用时，此命令将协议切换到指定版本，并接受以下选项：

* `AUTH <username> <password>`：直接在切换到指定协议版本的同时进行连接验证。这使得在建立新连接时，在调用`HELLO`之前调用`AUTH`变得不必要。请注意，`username`可以设置为"default"，以对不使用ACL，而是使用Redis版本6之前的简单的`requirepass`机制进行服务器身份验证。
* `SETNAME <clientname>`：相当于调用`CLIENT SETNAME`。
