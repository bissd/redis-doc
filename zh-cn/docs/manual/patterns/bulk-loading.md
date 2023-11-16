---
title: "Bulk loading"
linkTitle: "Bulk loading"
weight: 1
description: >
    使用Redis协议批量写入数据
aliases: [
    /topics/mass-insertion,
    /docs/reference/patterns/bulk-loading
]
---

批量加载是将大量预先存在的数据加载到Redis的过程。理想情况下，您希望快速高效地执行此操作。本文介绍了Redis中批量加载数据的一些策略。

## 使用Redis协议进行批量加载

使用普通的Redis客户端进行批量加载并不是一个好主意，原因有几个：采用逐个发送命令的朴素方法很慢，因为每个命令都要支付往返时间的开销。尽管可以使用流水线技术，但对于大量记录的批量加载，您需要在读取回复的同时编写新命令，以确保尽快插入数据。

只有少数客户端支持非阻塞I/O，并且并不是所有客户端都能以高效的方式解析回复以最大化吞吐量。出于所有这些原因，将数据批量导入Redis的首选方式是生成一个包含Redis协议的文本文件，以原始格式调用插入所需数据的命令。

例如，如果我需要生成一个包含数十亿个键的大型数据集，其形式为：`keyN -> ValueN'，我将创建一个包含以下命令的文件，按照Redis协议格式编写：

    SET Key0 Value0
    SET Key1 Value1
    ...
    SET KeyN ValueN

一旦创建此文件，其余操作就是尽快将其发送给Redis。
过去可用的方法是使用以下命令的`netcat`：

    (cat data.txt; sleep 10) | nc localhost 6379 > /dev/null

然而，这并不是一种非常可靠的批量导入方式，因为netcat并不知道所有数据何时传输完毕，并且无法检查错误。在Redis的2.6或更高版本中，`redis-cli`工具支持一种新模式，称为**管道模式**，旨在执行批量加载。

使用管道模式运行的命令如下所示：

    cat data.txt | redis-cli --pipe

这将产生类似于以下内容的输出：

    All data transferred. Waiting for the last reply...
    Last reply received from server.
    errors: 0, replies: 1000000

redis-cli实用程序还会确保只将从Redis实例接收到的错误重定向到标准输出。

### 生成 Redis 协议

Redis协议生成和解析非常简单，并且在[此处记录](/topics/protocol)。然而，为了生成用于批量加载的协议，您不需要完全了解协议的每个细节，只需知道每个命令以以下方式表示:

    *<args><cr><lf>
    $<len><cr><lf>
    <arg0><cr><lf>
    <arg1><cr><lf>
    ...
    <argN><cr><lf>

其中 `<cr>` 表示 "\r" （或 ASCII 字符 13），而 `<lf>` 表示 "\n" （或 ASCII 字符 10）。

例如，命令**SET key value**由以下协议表示：

    *3<cr><lf>
    $3<cr><lf>
    SET<cr><lf>
    $3<cr><lf>
    key<cr><lf>
    $5<cr><lf>
    value<cr><lf>

    "*3\r\n$3\r\nSET\r\n$3\r\nkey\r\n$5\r\nvalue\r\n"

您需要生成用于批量加载的文件只是由上述方式表示的命令依次组成。

以下是一个生成有效协议的 Ruby 函数：

    def gen_redis_proto(*cmd)
        proto = ""
        proto << "*"+cmd.length.to_s+"\r\n"
        cmd.each{|arg|
            proto << "$"+arg.to_s.bytesize.to_s+"\r\n"
            proto << arg.to_s+"\r\n"
        }
        proto
    end

    puts gen_redis_proto("SET","mykey","Hello World!").inspect

使用上述函数，可以轻松地生成上述示例中的键值对：
```python
    def generate_key_value_pairs():
        pairs = {
            "key1": "value1",
            "key2": "value2",
            "key3": "value3"
        }
        return pairs

    pairs = generate_key_value_pairs()
```

    (0...1000).each{|n|
        STDOUT.write(gen_redis_proto("SET","Key#{n}","Value#{n}"))
    }

我们可以直接在管道中运行程序到redis-cli，以执行我们的首次大规模导入会话。

    $ ruby proto.rb | redis-cli --pipe
    All data transferred. Waiting for the last reply...
    Last reply received from server.
    errors: 0, replies: 1000

### 管道模式的工作原理

在redis-cli的管道模式中所需的魔法是能够像netcat一样快速，同时仍然能够理解服务器发送的最后回复是什么时候发送的。

以下是获得的方式：

以下是所有数据，请将其视为正文HanziSimplified：

+ redis-cli --pipe 尝试尽可能快地将数据发送到服务器。
+ 同时，它会在有数据可读时尝试解析它。
+ 一旦从 stdin 中没有更多数据可读，它会发送一个带有 20 字节随机字符串的特殊 **ECHO** 命令：我们确定这是最后发送的命令，我们也确定我们可以通过检查是否接收到相同的 20 字节作为块回复来匹配回复。
+ 一旦发送了这个特殊的最后命令，接收回复的代码开始将回复与这些 20 个字节匹配。当达到匹配回复时，它可以成功退出。

使用这个技巧，我们不需要解析发送给服务器的协议，只需要解析回复，就能理解我们发送了多少个命令。

然而，在解析回复时，我们会计算解析的所有回复的计数器，以便最后能够告诉用户通过批量插入会话传输到服务器的命令数量。
