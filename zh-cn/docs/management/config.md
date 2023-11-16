---
title: "Redis配置"
linkTitle: "配置"
weight: 2
description: >
    redis.conf 的概述，Redis 的配置文件
aliases: [
    /docs/manual/config
    ]

---

Redis能够在没有配置文件的情况下使用内置的默认配置启动，但是这种设置只推荐用于测试和开发目的。

在配置Redis时，正确的方法是使用一个Redis配置文件，通常被称为`redis.conf`。

`redis.conf`文件包含一些非常简单的指令，其格式如下：

    keyword argument1 argument2 ... argumentN

这是一个配置指令的示例：

    replicaof 127.0.0.1 6380

可以使用（双引号或单引号）来提供包含空格的字符串作为参数，如下例所示：

    requirepass "hello world"

Single-quoted string can contain characters escaped by backslashes, and
double-quoted strings can additionally include any ASCII symbols encoded using
backslashed hexadecimal notation "\\xff".

单引号字符串可以包含通过反斜杠转义的字符，并且双引号字符串还可以使用通过反斜杠十六进制表示的任何ASCII符号"\\xff"。

配置指令列表以及其含义和使用方法在自述示例redis.conf中提供，该示例已打包到Redis发行版中。

* [Redis 7.2 的自注释 redis.conf](https://raw.githubusercontent.com/redis/redis/7.2/redis.conf)。
* [Redis 7.0 的自注释 redis.conf](https://raw.githubusercontent.com/redis/redis/7.0/redis.conf)。
* [Redis 6.2 的自注释 redis.conf](https://raw.githubusercontent.com/redis/redis/6.2/redis.conf)。
* [Redis 6.0 的自注释 redis.conf](https://raw.githubusercontent.com/redis/redis/6.0/redis.conf)。
* [Redis 5.0 的自注释 redis.conf](https://raw.githubusercontent.com/redis/redis/5.0/redis.conf)。
* [Redis 4.0 的自注释 redis.conf](https://raw.githubusercontent.com/redis/redis/4.0/redis.conf)。
* [Redis 3.2 的自注释 redis.conf](https://raw.githubusercontent.com/redis/redis/3.2/redis.conf)。
* [Redis 3.0 的自注释 redis.conf](https://raw.githubusercontent.com/redis/redis/3.0/redis.conf)。
* [Redis 2.8 的自注释 redis.conf](https://raw.githubusercontent.com/redis/redis/2.8/redis.conf)。
* [Redis 2.6 的自注释 redis.conf](https://raw.githubusercontent.com/redis/redis/2.6/redis.conf)。
* [Redis 2.4 的自注释 redis.conf](https://raw.githubusercontent.com/redis/redis/2.4/redis.conf)。

通过命令行传递参数
---

您还可以直接通过命令行传递Redis配置参数。
这对于测试目的非常有用。
以下是一个例子，它启动一个新的Redis实例，使用端口6380，
作为运行在127.0.0.1端口6379上的实例的一个副本。

    ./redis-server --port 6380 --replicaof 127.0.0.1 6379

传递给命令行的参数格式与redis.conf文件中使用的格式完全相同，但关键字前面加上了“--”。

请注意，内部会生成一个内存临时配置文件（可能会将用户传递的配置文件拼接在一起），其中参数会被翻译成redis.conf的格式。

在服务器运行时更改Redis配置

可以在不停止和重新启动服务的情况下重新配置Redis，也可以使用特殊命令`CONFIG SET`和`CONFIG GET`以编程方式查询当前配置。

并非所有的配置指令都可以以这种方式支持，但大多数指令都按预期进行支持。
更多信息请参考 `CONFIG SET` 和 `CONFIG GET` 页面。

注意，即使在运行时修改配置也**不会影响redis.conf文件**，因此在下次重新启动Redis时将使用旧配置。

请确保根据使用`CONFIG SET`设置的配置修改`redis.conf`文件。
您可以手动修改，也可以使用`CONFIG REWRITE`，它会自动扫描您的`redis.conf`文件并更新与当前配置值不匹配的字段。
不存在但设置为默认值的字段不会被添加。
配置文件中的注释将被保留。

将Redis配置为缓存
---

如果您计划将Redis用作具有过期设置的缓存，您可以考虑使用以下配置（假设最大内存限制为2兆字节作为示例）：

    maxmemory 2mb
    maxmemory-policy allkeys-lru

在该配置中，应用程序无需使用`EXPIRE`命令（或等效命令）为键设置存活时间，因为只要我们达到2兆字节的内存限制，所有键将使用近似的LRU算法被清除。

基本上，在这个配置中，Redis的作用类似于memcached。
我们在这里有更详细的有关将Redis用作LRU缓存的文档。
