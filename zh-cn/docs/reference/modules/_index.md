---
title: "Redis模块API"
linkTitle: "模块API"
weight: 2
description: >
    Redis模块编写简介
aliases:
    - /topics/modules-intro
---

模块文档由以下页面组成:

* Redis模块介绍（此文件）。关于Redis模块系统和API的概述。建议从这里开始阅读。
* [实现原生数据类型](/topics/modules-native-types)介绍了将原生数据类型引入模块的实现方式。
* [阻塞操作](/topics/modules-blocking-ops)演示了如何编写阻塞命令，这些命令不会立即回复，但会阻塞客户端，而不会阻塞Redis服务器，并在可能的情况下提供回复。
* [Redis模块API参考](/topics/modules-api-ref)是从module.c RedisModule函数的顶部注释生成的。它是一个很好的参考，可以理解每个函数的工作原理。

Redis模块使得可以使用外部模块扩展Redis功能，快速实现类似于核心本身的新的Redis命令功能。

Redis模块是可以在Redis启动时加载或使用`MODULE LOAD`命令加载的动态库。Redis导出了一个C API，以一个名为`redismodule.h`的单个C头文件的形式。模块是用C语言编写的，然而也可以使用具有C绑定功能的C++或其他语言。

模块被设计成可以加载到不同版本的Redis中，
因此给定的模块不需要为了与特定版本的Redis一起运行而被设计或重新编译。
因此，该模块将使用特定的API版本向Redis核心注册。
当前的API版本是"1"。

这个文档是关于Redis模块的Alpha版本的。API、功能和其他细节将来可能会发生变化。

## 载入模块

为了测试您正在开发的模块，您可以使用以下 `redis.conf` 配置指令加载模块：

    loadmodule /path/to/mymodule.so

还可以使用以下命令在运行时加载模块：

    MODULE LOAD /path/to/mymodule.so

为了列出所有已加载的模块，请使用：

    MODULE LIST

最后，您可以使用以下命令卸载（以后如果需要，可以重新加载）一个模块：

    MODULE UNLOAD mymodule

请注意上面的`mymodule`不是没有`.so`后缀的文件名，而是模块用于向Redis核心注册自己的名称。可以使用`MODULE LIST`获取此名称。然而，最佳实践是动态库的文件名与模块向Redis核心注册自己的名称相同。

## 你可以编写的最简单的模块

为了展示模块的不同部分，这里将展示一个非常简单的模块，该模块实现了一个输出随机数的命令。

    #include "redismodule.h"
```c
#include <stdlib.h>
```

    int HelloworldRand_RedisCommand(RedisModuleCtx *ctx, RedisModuleString **argv, int argc) {
        RedisModule_ReplyWithLongLong(ctx,rand());
        return REDISMODULE_OK;
    }

    int RedisModule_OnLoad(RedisModuleCtx *ctx, RedisModuleString **argv, int argc) {
        if (RedisModule_Init(ctx,"helloworld",1,REDISMODULE_APIVER_1)
            == REDISMODULE_ERR) return REDISMODULE_ERR;

        if (RedisModule_CreateCommand(ctx,"helloworld.rand",
            HelloworldRand_RedisCommand, "fast random",
            0, 0, 0) == REDISMODULE_ERR)
            return REDISMODULE_ERR;

        return REDISMODULE_OK;
    }

示例模块具有两个函数。其中一个实现了一个名为HELLOWORLD.RAND的命令。这个函数是特定于该模块的。然而，另一个名为“RedisModule_OnLoad（）”的函数必须存在于每个Redis模块中。这是模块初始化的入口点，用于注册其命令和可能使用的其他私有数据结构。

注意：对于模块来说，用模块的名称加上一个点，然后是命令的名称是一个好主意，就像`HELLOWORLD.RAND`的情况一样。这样能够减少冲突的可能性。

请注意，如果不同的模块有冲突的命令，它们将无法同时在Redis中工作，因为`RedisModule_CreateCommand`函数将在其中一个模块中失败，导致模块加载中止并返回错误条件。

## 模块初始化

上述示例展示了函数`RedisModule_Init()`的用法。
它应该是模块`OnLoad`函数调用的第一个函数。
以下是该函数的原型：

    int RedisModule_Init(RedisModuleCtx *ctx, const char *modulename,
                         int module_version, int api_version);

`Init` 函数向 Redis 核心宣告了模块的名称、版本（由 `MODULE LIST` 报告）以及它愿意使用的特定 API 版本。

如果API版本错误、名称已被占用或存在其他类似的错误，则该函数将返回`REDISMODULE_ERR`，并且模块的`OnLoad`函数应立即返回错误。

在调用`Init`函数之前，不能调用其他任何API函数，否则模块将崩溃并导致Redis实例崩溃。

第二个函数叫做`RedisModule_CreateCommand`，用于将命令注册到Redis核心中。其原型如下：

    int RedisModule_CreateCommand(RedisModuleCtx *ctx, const char *name,
                                  RedisModuleCmdFunc cmdfunc, const char *strflags,
                                  int firstkey, int lastkey, int keystep);

正如您所看到的，大多数Redis模块API调用都将`context`作为第一个参数，以便它们可以引用调用它的模块、执行给定命令的命令和客户端等等。

要创建一个新的命令，上述函数需要上下文、命令的名称、指向实现命令的函数的指针、命令的标志以及命令参数中关键名称的位置。

执行命令的函数必须具有以下原型：

    int mycommand(RedisModuleCtx *ctx, RedisModuleString **argv, int argc);

命令函数参数只是上下文，将传递给所有其他 API 调用，命令参数向量以及由用户传递的参数总数。

正如您所见，参数以指向特定数据类型`RedisModuleString`的指针形式提供。这是一个不透明的数据类型，您可以使用API函数来访问和使用它，不需要直接访问其字段。

Zooming into the example command implementation, we can find another call:

放大到示例命令实现中，我们可以找到另一个调用：

    int RedisModule_ReplyWithLongLong(RedisModuleCtx *ctx, long long integer);

该函数返回一个整数给调用命令的客户端，与其他Redis命令的行为完全相同，例如`INCR`或`SCARD`。

## 模块清理

在大多数情况下，没有必要进行特殊的清理。
当一个模块被卸载时，Redis将自动注销命令和取消订阅通知。
然而，在一个模块包含持久化内存或配置的情况下，模块可能包含一个可选的RedisModule_OnUnload函数。
如果一个模块提供了这个函数，它将在模块卸载过程中被调用。
以下是函数原型：

    int RedisModule_OnUnload(RedisModuleCtx *ctx);

`OnUnload`函数可能会通过返回`REDISMODULE_ERR`来阻止模块的卸载。否则，应返回`REDISMODULE_OK`。

## Redis 模块的设置和依赖

Redis模块不依赖于Redis或其他库，并且它们不需要使用特定的`redismodule.h`文件进行编译。要创建一个新模块，只需将最新版本的`redismodule.h`复制到源代码树中，连接所有所需的库，并创建一个导出`RedisModule_OnLoad()`函数符号的动态库。

该模块将能够加载到不同版本的Redis中。

可以设计一个模块来支持较新和较旧的Redis版本，在某些版本中并不支持某些API函数。
如果在当前运行的Redis版本中未实现某个API函数，函数指针将被设置为NULL。
这样可以让模块在使用函数之前检查它是否存在：

    if (RedisModule_SetCommandInfo != NULL) {
        RedisModule_SetCommandInfo(cmd, &info);
    }

在最近的`redismodule.h`版本中，定义了一个方便的宏`RMAPI_FUNC_SUPPORTED(funcname)`。
使用这个宏或者只是与NULL进行比较是个人喜好的问题。

# 传递配置参数给Redis模块

当使用`MODULE LOAD`命令或在`redis.conf`文件中使用`loadmodule`指令加载模块时，用户可以通过在模块文件名后添加参数来向模块传递配置参数。

    loadmodule mymodule.so foo bar 1234

在上面的示例中，字符串`foo`，`bar`和`1234`将作为RedisModuleString指针数组的参数`argv`传递给模块的`OnLoad()`函数。传递的参数个数是`argc`。

通常，模块将以一些`static`全局变量存储模块配置参数，这些参数可以在整个模块中访问，因此可以通过更改配置来改变不同命令的行为。

## 使用RedisModuleString对象工作

命令参数向量`argv`传递给模块命令以及其他模块API函数的返回值都是`RedisModuleString`类型。

通常情况下，您可以直接将模块字符串传递给其他API调用，然而有时您可能需要直接访问字符串对象。

有几个函数可以用来处理字符串对象：

    const char *RedisModule_StringPtrLen(RedisModuleString *string, size_t *len);

上述函数通过返回指针和设置长度来访问字符串。
从`const`指针限定符可以看出，您不应该对字符串对象指针进行写操作。

但是，如果你愿意的话，你可以使用以下API来创建新的字符串对象：

    RedisModuleString *RedisModule_CreateString(RedisModuleCtx *ctx, const char *ptr, size_t len);

上述命令返回的字符串必须使用相应的`RedisModule_FreeString()`调用进行释放:

    void RedisModule_FreeString(RedisModuleString *str);

然而，如果您想避免释放字符串的操作，本文档后面介绍的自动内存管理机制可以是一个不错的选择，它可以为您完成这个操作。

请注意，通过参数向量`argv`提供的字符串不需要释放。您只需要释放您创建的新字符串或由其他API返回的新字符串，其中特定说明返回的字符串必须被释放。

## 从数字创建字符串或将字符串解析为数字

创建一个新的字符串来自一个整数是非常常见的操作，所以有一个函数可以做到这一点：

    RedisModuleString *mystr = RedisModule_CreateStringFromLongLong(ctx,10);

同样，为了将字符串解析为数字：

    long long myval;
    if (RedisModule_StringToLongLong(ctx,argv[1],&myval) == REDISMODULE_OK) {
        /* Do something with 'myval' */
    }

## 从模块中访问 Redis 键

大多数Redis模块为了有用，必须与Redis数据空间进行交互（这并不总是正确的，例如ID生成器可能永远不会触碰Redis键）。为了访问Redis数据空间，Redis模块有两个不同的API，一个是低级API，提供非常快的访问速度和一组函数来操作Redis数据结构。另一个API更高级，允许调用Redis命令并获取结果，类似于Lua脚本访问Redis。

高级API在访问Redis功能时也很有用，这些功能在API中不可用。

通常，模块开发者应该偏好低级API，因为使用低级API实现的命令的运行速度与本机Redis命令的速度相当。然而，高级API确实有一些使用案例。例如，瓶颈往往可能是数据的处理而不是访问数据。

有时候使用低级别的API并不比使用高级别的API困难。

## 调用Redis命令

访问Redis的高级API是`RedisModule_Call()`函数与访问`Call()`返回的回复对象所需的函数的总和。

`RedisModule_Call` 使用一种特殊的调用约定，它使用格式说明符来指定将什么类型的对象作为参数传递给函数。

Redis 命令只需要使用命令名和参数列表来调用。然而，在调用命令时，参数可能来自不同类型的字符串：以空字符结尾的 C 字符串，从命令实现中的 `argv` 参数接收的 RedisModuleString 对象，带有指针和长度的二进制安全 C 缓冲区等等。

例如，如果我想要调用`INCRBY`函数，使用一个参数（键）
在参数向量`argv`中接收到的字符串，该向量是RedisModuleString对象指针的数组，
以及一个表示数字"10"的C字符串作为第二个参数（增量），我将使用以下函数调用：

    RedisModuleCallReply *reply;
    reply = RedisModule_Call(ctx,"INCRBY","sc",argv[1],"10");

第一个参数是上下文，第二个参数始终是以空字符结尾的C字符串，表示命令名称。第三个参数是格式说明符，其中每个字符对应后续的参数类型。在上述情况中， `"sc"` 表示一个 RedisModuleString 对象和一个以空字符结尾的C字符串。其他参数只是按照规定的两个参数赋值。实际上，`argv[1]` 是一个 RedisModuleString 对象，`"10"` 是一个以空字符结尾的C字符串。

以下是格式化说明符的完整列表：

`%d`：带符号的十进制整数
`%i`：带符号的十进制、八进制或十六进制整数
`%u`：无符号十进制整数
`%f`：十进制浮点数
`%e`：科学计数法表示的浮点数
`%g`：根据数值不同自动选择`%f`或`%e`
`%c`：单个字符
`%s`：字符串
`%p`：指针地址
`%n`：已写字符数的指针
`%x`：带符号的十六进制整数（小写字母）
`%X`：带符号的十六进制整数（大写字母）
`%o`：八进制整数
`%%`：显示百分号

以下是常用的格式化说明符的选项：
`%10d`：最小宽度为10的带符号十进制整数
`%.2f`：精度为2的十进制浮点数
`%-10s`：左对齐最小宽度为10的字符串
`%+d`：显示符号的带符号十进制整数
`%05d`：最小宽度为5的带符号十进制整数（用0进行填充）
`%#x`：显示带有前缀的带符号十六进制整数

请注意：以上仅为常见的格式化说明符和选项，具体使用时还可以根据实际需要进行更多的调整和组合。

* **c**  - 空终止的C字符串指针。
* **b**  - C缓冲区，需要两个参数：C字符串指针和`size_t`长度。
* **s**  - 以`argv`接收到或通过其他返回`RedisModuleString`对象的Redis模块API接收到的RedisModuleString。
* **l**  - 长整型整数。
* **v**  - RedisModuleString对象的数组。
* **!**  - 此修饰符仅告诉函数将命令复制到副本和AOF。从参数解析的角度来看，它被忽略。
* **A**  - 当给定`!`时，该修饰符告诉抑制AOF传播：命令将仅传播到副本。
* **R**  - 当给定`!`时，该修饰符告诉抑制副本传播：命令将仅传播到启用的AOF。

该函数在成功时返回一个 `RedisModuleCallReply` 对象，在错误时返回 NULL。

当命令名称无效时返回NULL，当格式限定符使用无法识别的字符时，或当命令使用错误数量的参数调用时也返回NULL。在上述情况下，errno变量将设置为EINVAL。当启用集群的实例中目标键不在本地哈希槽中时，也会返回NULL。在这种情况下，errno将被设置为EPERM。

## 使用 RedisModuleCallReply 对象进行操作。

`RedisModuleCall`返回的是通过`RedisModule_CallReply*`函数族访问的回复对象。

为了获取类型或回复（对应于Redis协议支持的其中一种数据类型），使用函数`RedisModule_CallReplyType()`：

    reply = RedisModule_Call(ctx,"INCRBY","sc",argv[1],"10");
    if (RedisModule_CallReplyType(reply) == REDISMODULE_REPLY_INTEGER) {
        long long myval = RedisModule_CallReplyInteger(reply);
        /* Do something with myval. */
    }

有效的回复类型包括:

* `REDISMODULE_REPLY_STRING` 批量字符串或状态回复。
* `REDISMODULE_REPLY_ERROR` 错误。
* `REDISMODULE_REPLY_INTEGER` 有符号的64位整数。
* `REDISMODULE_REPLY_ARRAY` 回复的数组。
* `REDISMODULE_REPLY_NULL` 空回复。

字符串、错误和数组都有一个关联的长度。对于字符串和错误，长度对应于字符串的长度。对于数组，长度是元素的数量。要获取回复的长度，使用以下函数：

    size_t reply_len = RedisModule_CallReplyLength(reply);

为了获取整数回复的值，使用如上面示例中已经显示的以下函数：

    long long reply_integer_val = RedisModule_CallReplyInteger(reply);

调用错误类型的回复对象时，上述函数始终返回`LLONG_MIN`。

数组回复的子元素可以通过以下方式访问：

    RedisModuleCallReply *subreply;
    subreply = RedisModule_CallReplyArrayElement(reply,idx);

如果您尝试访问超出范围的元素，则上述函数将返回NULL。

使用以下方式可以访问字符串和错误（类似于字符串但具有不同的类型），确保永远不要向返回的指针写入数据（该指针返回为`const`指针，以便错误使用必须显式说明）：

    size_t len;
    char *ptr = RedisModule_CallReplyStringPtr(reply,&len);

如果答复类型不是字符串或错误，则返回NULL。

RedisCallReply对象与模块字符串对象（RedisModuleString类型）不同。但是有时您可能需要将字符串类型或整数类型的回复传递给期望模块字符串的API函数。

当出现这种情况时，您可能希望评估是否使用低级API可以更简单地实现您的命令，或者您可以使用以下函数从字符串、错误或整数类型的调用回复中创建一个新的字符串对象。

    RedisModuleString *mystr = RedisModule_CreateStringFromCallReply(myreply);

如果回复的类型不正确，则返回NULL。
返回的字符串对象应该使用`RedisModule_FreeString()`释放，
或者通过启用自动内存管理（参见相应部分）来释放。

## 释放调用回复对象

回复对象必须使用`RedisModule_FreeCallReply`进行释放。对于数组，只需要释放顶层的回复对象，不需要释放嵌套的回复对象。
目前的模块实现提供了一种保护机制，以避免在释放错误的嵌套回复对象时崩溃，然而这个特性不能保证永远存在，所以不应该视为API的一部分。

如果您使用自动内存管理（稍后在本文档中解释），您不需要释放回复（但如果您希望释放内存，仍然可以释放）。

## 从Redis命令返回值

像普通的Redis命令一样，通过模块实现的新命令必须能够将值返回给调用者。为了实现这个目标，API导出了一组函数，用于返回Redis协议的常规类型以及这些类型的数组元素。还可以返回带有任何错误字符串和代码的错误（错误代码是错误消息中的首字母大写，比如错误消息"BUSY the server is busy"中的"BUSY"字符串）。

所有向客户端发送回复的函数都被称为`RedisModule_ReplyWith<something>`。

要返回一个错误，请使用：

    RedisModule_ReplyWithError(RedisModuleCtx *ctx, const char *err);

对于错误的键类型，有一个预定义的错误字符串：

    REDISMODULE_ERRORMSG_WRONGTYPE

示例用法：

```python
python translate.py --text "Hello, how are you?" --source en --target zh-CN
```

```bash
./translate.sh --text "你好，你们好吗？" --source zh-CN --target en
```

| Option | Description |
|--------|-------------|
| `--text` | The text to be translated |
| `--source` | The source language |
| `--target` | The target language |

    RedisModule_ReplyWithError(ctx,"ERR invalid arguments");

我们已经在上面的例子中看到了如何用`long long`回复。

    RedisModule_ReplyWithLongLong(ctx,12345);

为了以简单字符串的形式回复，这个字符串不能包含二进制值或换行符，
（所以适合发送类似"OK"的小单词），我们使用：

    RedisModule_ReplyWithSimpleString(ctx,"OK");

根据不同的函数，可以使用“大块字符串”进行二进制安全的回复。

    int RedisModule_ReplyWithStringBuffer(RedisModuleCtx *ctx, const char *buf, size_t len);

    int RedisModule_ReplyWithString(RedisModuleCtx *ctx, RedisModuleString *str);

第一个函数接受一个C指针和长度。第二个函数接受一个RedisModuleString对象。根据手头的源类型使用其中之一。

为了用一个数组作为回复，你只需要使用一个函数来发射数组的长度，然后调用上述函数的次数与数组元素数量相同。

    RedisModule_ReplyWithArray(ctx,2);
    RedisModule_ReplyWithStringBuffer(ctx,"age",3);
    RedisModule_ReplyWithLongLong(ctx,22);

返回嵌套数组很容易，您的嵌套数组元素只需使用另一个`RedisModule_ReplyWithArray()`调用，然后再调用发射子数组元素。

## 返回具有动态长度的数组

有时候无法事先知道数组中的项目数量。例如，考虑一个实现了FACTOR命令的Redis模块，它可以输入一个数字并输出其质因数。与计算阶乘、将质因数存储到数组中，然后再生成命令回复的方法相比，更好的解决方案是先创建一个长度未知的数组回复，然后再设置其长度。这可以通过`RedisModule_ReplyWithArray()`的一个特殊参数来实现：

    RedisModule_ReplyWithArray(ctx, REDISMODULE_POSTPONED_LEN);

上述调用启动了一个数组回复，因此我们可以使用其他`ReplyWith`调用来生成数组项。最后，为了设置长度，请使用以下调用：

    RedisModule_ReplySetArrayLength(ctx, number_of_items);

在 FACTOR 命令的情况下，这将转换为类似以下代码的代码：

    RedisModule_ReplyWithArray(ctx, REDISMODULE_POSTPONED_LEN);
    number_of_factors = 0;
    while(still_factors) {
        RedisModule_ReplyWithLongLong(ctx, some_factor);
        number_of_factors++;
    }
    RedisModule_ReplySetArrayLength(ctx, number_of_factors);

另一个常见的用例是遍历某个集合的数组，并只返回满足某种筛选条件的元素。

在回复中允许有多个嵌套数组。
每次调用 `SetArray()` 都会设置最新的对应调用 `ReplyWithArray()` 的长度：

    RedisModule_ReplyWithArray(ctx, REDISMODULE_POSTPONED_LEN);
    ... generate 100 elements ...
    RedisModule_ReplyWithArray(ctx, REDISMODULE_POSTPONED_LEN);
    ... generate 10 elements ...
    RedisModule_ReplySetArrayLength(ctx, 10);
    RedisModule_ReplySetArrayLength(ctx, 100);

这将创建一个包含100个项的数组，最后一个元素是一个包含10个项的数组。

## 参数数量和类型检查

通常命令需要检查参数的数量和键的类型是否正确。为了报告参数错误，可以使用特定的函数`RedisModule_WrongArity()`。使用方法很简单：

    if (argc != 2) return RedisModule_WrongArity(ctx);

检查错误类型涉及打开键并检查类型：

    RedisModuleKey *key = RedisModule_OpenKey(ctx,argv[1],
        REDISMODULE_READ|REDISMODULE_WRITE);

    int keytype = RedisModule_KeyType(key);
    if (keytype != REDISMODULE_KEYTYPE_STRING &&
        keytype != REDISMODULE_KEYTYPE_EMPTY)
    {
        RedisModule_CloseKey(key);
        return RedisModule_ReplyWithError(ctx,REDISMODULE_ERRORMSG_WRONGTYPE);
    }

要注意的是，通常情况下，如果键是预期类型或为空，您都希望继续使用命令。

## 低级别访问键

低级别的键访问允许直接对与键相关联的值对象执行操作，速度类似于Redis在内部实现内置命令的速度。

一旦打开一个键，将返回一个键指针，该指针可用于与所有其他低级API调用一起执行对键或其关联值的操作。

由于API的目标是速度非常快，它不能进行太多的运行时检查，因此用户必须了解并遵守某些规则：

* 同时打开至少一个用于写入的实例的同一个键多次是未定义的，可能会导致崩溃。
* 在键已经打开的情况下，只能通过低级键API访问它。例如，打开一个键，然后使用`RedisModule_Call()` API调用`DEL`命令会导致崩溃。然而，可以安全地打开一个键，使用低级API执行一些操作，然后关闭它，再使用其他API来管理同一个键，然后再次打开它进行更多操作。

为了打开一个键，使用`RedisModule_OpenKey`函数。它返回一个键指针，我们将使用这个指针来访问和修改值的所有后续调用：

    RedisModuleKey *key;
    key = RedisModule_OpenKey(ctx,argv[1],REDISMODULE_READ);

第二个参数是键名，必须是`RedisModuleString`对象。
第三个参数是模式：`REDISMODULE_READ`或`REDISMODULE_WRITE`。
可以使用`|`将两个模式进行按位或运算，以同时在两种模式下打开键。目前，以写模式打开的键也可以以读模式访问，但这应被视为实现细节。在合理的模块中应使用正确的模式。

您可以打开不存在的键进行写入，因为在对键进行写入尝试时，键将被创建。然而，当仅用于阅读时，如果键不存在，`RedisModule_OpenKey`将返回NULL。

使用完一个键后，您可以使用以下命令进行关闭：
```python
client.close()
```

    RedisModule_CloseKey(key);

请注意，如果启用了自动内存管理，则不需要强制关闭键。当模块函数返回时，Redis会负责关闭所有仍然打开的键。

## 获取密钥类型

为了获取键的值，请使用`RedisModule_KeyType()`函数：

    int keytype = RedisModule_KeyType(key);

它返回以下值之一：

    REDISMODULE_KEYTYPE_EMPTY
    REDISMODULE_KEYTYPE_STRING
    REDISMODULE_KEYTYPE_LIST
    REDISMODULE_KEYTYPE_HASH
    REDISMODULE_KEYTYPE_SET
    REDISMODULE_KEYTYPE_ZSET

上述仅是常见的Redis键类型，除了一个空类型，表示键指针关联到一个不存在的空键。

## 创建新的密钥

为了创建一个新的键，打开它以便写入，并使用其中一个键写入函数进行写入。示例：

    RedisModuleKey *key;
    key = RedisModule_OpenKey(ctx,argv[1],REDISMODULE_WRITE);
    if (RedisModule_KeyType(key) == REDISMODULE_KEYTYPE_EMPTY) {
        RedisModule_StringSet(key,argv[2]);
    }

## 删除键

只是使用：

    RedisModule_DeleteKey(key);

如果键未打开以进行写入，则该函数返回`REDISMODULE_ERR`。
请注意，在键被删除后，它会被设置为可以被新键命令定位。
例如，`RedisModule_KeyType()`将返回它是一个空键，并且对其进行写入将创建一个新键，可能是另一种类型（具体取决于所使用的API）。

## 管理键过期时间 (TTLs)

为了控制关键字的过期时间，提供了两个函数，可以设置、修改、获取和取消与关键字关联的生存时间。

为了查询现有密钥的当前到期时间，使用以下一段函数：

    mstime_t RedisModule_GetExpire(RedisModuleKey *key);

该函数以毫秒为单位返回键的存活时间，如果键没有关联的过期时间或者根本不存在，则返回特殊值`REDISMODULE_NO_EXPIRE`（通过检查键的类型是否为`REDISMODULE_KEYTYPE_EMPTY`来区分这两种情况）。

为了更改密钥的过期时间，使用以下函数：

    int RedisModule_SetExpire(RedisModuleKey *key, mstime_t expire);

当在一个不存在的键上调用时，会返回`REDISMODULE_ERR`，因为该函数只能将过期时间关联到已存在的打开键（不存在的打开键只有在使用特定数据类型的写操作创建新值时才有用）。

`expire`时间以毫秒为单位指定。如果键目前没有过期时间，则设置新的过期时间。如果键已经有过期时间，则用新值替换。

如果键具有到期时间，并且使用特殊值`REDISMODULE_NO_EXPIRE`作为新到期时间，那么到期时间将被移除，类似于Redis的`PERSIST`命令。如果键已经是持久化的，则不执行任何操作。

## 获取值的长度

有一个单一函数来检索与一个打开的键相关联的值的长度。返回的长度是值特定的，对于字符串是字符串的长度，对于聚合数据类型（如列表、集合、排序集和哈希表）是元素的数量。

    size_t len = RedisModule_ValueLength(key);

如果键不存在，则函数返回0：

## 字符串类型 API

设置一个新的字符串值，就像Redis的`SET`命令一样，可以使用以下方式进行操作：

    int RedisModule_StringSet(RedisModuleKey *key, RedisModuleString *str);

这个函数的工作原理与 Redis 的 `SET` 命令完全相同，即如果存在先前的值（无论其类型如何），将被删除。

使用DMA（直接内存访问）来提高速度，可以访问现有字符串值。API将返回一个指针和长度，从而可以直接访问字符串，并在需要时修改。

    size_t len, j;
    char *myptr = RedisModule_StringDMA(key,&len,REDISMODULE_WRITE);
    for (j = 0; j < len; j++) myptr[j] = 'A';

在上面的例子中，我们直接在字符串上进行写入操作。请注意，如果您想要进行写入操作，必须确保要求使用“WRITE”模式。

DMA指针仅在在使用指针进行DMA调用之后，在与该键进行其他操作之前有效。

有时候当我们想直接操作字符串时，我们还需要改变它们的大小。为了实现这个目的，我们可以使用`RedisModule_StringTruncate`函数。示例：

    RedisModule_StringTruncate(mykey,1024);

函数根据需要截断或扩大字符串，如果先前的长度小于我们请求的新长度，则用零字节对其进行填充。
如果字符串不存在，因为`key`与一个开放的空键关联，将创建一个字符串值并与该键关联。

注意每次调用 `StringTruncate()` 函数时，我们需要重新获取 DMA 指针，因为旧指针可能无效。

## 列表类型的API

可以从列表值中推入和弹出数值：

    int RedisModule_ListPush(RedisModuleKey *key, int where, RedisModuleString *ele);
    RedisModuleString *RedisModule_ListPop(RedisModuleKey *key, int where);

在两个API中，`where`参数指定是从尾部还是头部进行推送或弹出，使用以下宏定义：

    REDISMODULE_LIST_HEAD
    REDISMODULE_LIST_TAIL

`RedisModule_ListPop()` 返回的元素类似使用 `RedisModule_CreateString()` 创建的字符串，必须使用 `RedisModule_FreeString()` 或启用自动内存管理来释放。

## 设置类型 API

进行中的工作。

## Sorted set 类型 API

文档缺失，请参考`module.c`中的顶部注释来了解以下函数的信息：

* `RedisModule_ZsetAdd`
* `RedisModule_ZsetIncrby`
* `RedisModule_ZsetScore`
* `RedisModule_ZsetRem`

对于已排序集合迭代器：

* `RedisModule_ZsetRangeStop`
* `RedisModule_ZsetFirstInScoreRange`
* `RedisModule_ZsetLastInScoreRange`
* `RedisModule_ZsetFirstInLexRange`
* `RedisModule_ZsetLastInLexRange`
* `RedisModule_ZsetRangeCurrentElement`
* `RedisModule_ZsetRangeNext`
* `RedisModule_ZsetRangePrev`
* `RedisModule_ZsetRangeEndReached`

## Hash类型的API

文档缺失，请查阅 `module.c` 中的顶部注释，了解以下函数的说明：

* `RedisModule_HashSet`
* `RedisModule_HashGet`

## 迭代聚合值

正在进行中的工作。

## 复制命令

如果您想要在复制的Redis实例的上下文中，或者通过使用AOF文件进行持久化，像普通的Redis命令一样使用模块命令时，保持一致的复制处理对于模块命令来说是非常重要的。

在使用更高级别的API调用命令时，如果在`RedisModule_Call()`的格式字符串中使用"!"修饰符，复制将自动发生，如以下示例所示：

    reply = RedisModule_Call(ctx,"INCRBY","!sc",argv[1],"10");

正如您所看到的格式说明符为`"!sc"`。叹号不会被解析为格式说明符，但它会内部标记该命令为“必须复制”。

如果您使用以上的编程风格，那么就没有问题。
然而有时候情况比较复杂，您可能需要使用底层 API。
在这种情况下，如果命令执行没有任何副作用，并且一直执行同样的工作，
那么可以将用户执行的命令原封不动地复制出来。要实现这个功能，
只需要调用以下函数：

    RedisModule_ReplicateVerbatim(ctx);

使用上述API时，不应使用任何其他复制功能，因为不能保证其兼容性。

然而，这不是唯一的选择。也可以通过一个类似于`RedisModule_Call()`的API，准确地告诉Redis要复制哪些命令作为命令执行的效果，而不是调用命令，将其发送到AOF/副本流。举个例子：

    RedisModule_Replicate(ctx,"INCRBY","cl","foo",my_increment);

可以多次调用`RedisModule_Replicate`，每次调用都会发出一个命令。所有发出的序列都被包装在`MULTI/EXEC`事务之间，使得AOF和复制效果与执行单个命令相同。

请注意，`Call()`复制和`Replicate()`复制有一个规则，
如果您想混合使用这两种复制形式（如果有更简单的方法，这不一定是个好主意）。
使用`Call()`复制的命令总是在最终的`MULTI/EXEC`块中首先被发出，而使用`Replicate()`发出的所有命令将会跟随其后。

## 自动内存管理

通常在使用C语言编写程序时，程序员需要手动管理内存。这就是为什么Redis模块API具有释放字符串、关闭打开的键、释放回复等功能的原因。

然而，由于命令在一个封闭的环境中执行，并且具有严格的API集合，Redis能够为模块提供自动内存管理，在一定程度上会牺牲一些性能（大多数情况下，这是非常低的代价）。

启用自动内存管理时：

1. 您不需要关闭打开的键。
2. 您不需要释放回复。
3. 您不需要释放 RedisModuleString 对象。

不过如果你愿意的话，你仍然可以这样做。例如，在循环中分配大量字符串时，自动内存管理可能是激活的，但你可能仍然希望释放不再使用的字符串。

为了启用自动内存管理，只需在命令实现的开始调用以下函数：

    RedisModule_AutoMemory(ctx);

通常情况下，自动内存管理是一个好的选择，但有经验的 C 程序员可能不会使用它，以获得一些速度和内存使用方面的好处。

## 将内存分配到模块中

普通的C程序使用`malloc()`和`free()`来动态分配和释放内存。虽然在Redis模块中使用`malloc`技术上不是被禁止的，但更好的做法是使用Redis模块特定的函数，这些函数是`malloc`、`free`、`realloc`和`strdup`的精确替代品。这些函数包括：

    void *RedisModule_Alloc(size_t bytes);
    void* RedisModule_Realloc(void *ptr, size_t bytes);
    void RedisModule_Free(void *ptr);
    void RedisModule_Calloc(size_t nmemb, size_t size);
    char *RedisModule_Strdup(const char *str);

以下的标记语言文本会与它们的libc等效函数一样工作，不过它们使用的是Redis相同的分配器，并且使用这些函数分配的内存会在`INFO`命令的内存部分报告，也在实施`maxmemory`策略时进行核算，通常是Redis可执行文件的核心部分。相对而言，在模块中使用libc的`malloc()`分配的内存对Redis来说是透明的。

使用模块函数来分配内存的另一个原因是，在模块内部创建本地数据类型时，RDB加载函数可以直接返回反序列化的字符串（从RDB文件中），作为`RedisModule_Alloc()`分配的内存，因此可以直接用于加载后的数据结构填充，而无需将它们复制到数据结构中。

# 内存池分配器

有时在命令实现中，需要执行许多小的分配，这些分配在命令执行结束时将不会保留，只是用于执行命令本身的功能。

使用Redis池分配器可以更轻松地完成这项工作：

    void *RedisModule_PoolAlloc(RedisModuleCtx *ctx, size_t bytes);

这个函数的工作方式类似于`malloc()`，并返回对齐到大于或等于`bytes`的下一个二的幂的内存（最大对齐为8字节）。然而，它是以块的形式分配内存，所以分配的开销很小，更重要的是，在命令返回时分配的内存会自动释放。

总之，短期使用的分配是池分配器的良好选择候选者。

## 编写与Redis集群兼容的命令

文档缺失，请检查`module.c`内的以下函数：

    RedisModule_IsKeysPositionRequest(ctx);
    RedisModule_KeyAtPos(ctx,pos);
