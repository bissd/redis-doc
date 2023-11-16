---
title: "本文介绍本地类型的模块API。"
linkTitle: "原生类型API"
weight: 1
description: >
    如何在Redis模块中使用本机类型
aliases:
    - /topics/modules-native-types
---

Redis模块可以在高级别通过调用Redis命令访问Redis内置的数据结构，也可以在低级别直接操作数据结构。

使用这些功能来在现有的Redis数据结构上构建新的抽象，或者使用字符串DMA来编码模块数据结构到Redis字符串中，可以创建出感觉像是导出新数据类型的模块。然而，对于更复杂的问题，这还不够，需要在模块内部实现新的数据结构。

我们称Redis模块能够实现感觉像原生Redis数据结构的新数据结构为"原生类型支持"。本文档描述了Redis模块系统导出的API，以创建新的数据
结构，在RDB文件中处理序列化，在AOF中进行重写过程，在`TYPE`命令中报告类型等。

原生类型概览
---

一个导出本机类型的模块由以下主要部分组成：

* 某种新数据结构的实现和对新数据结构的操作命令。
* 处理RDB保存、RDB加载、AOF重写、释放与键关联的值、计算与`DEBUG DIGEST`命令一起使用的值摘要（哈希）的一组回调函数。
* 一个9个字符的名称，对每个模块本地数据类型是唯一的。
* 一个编码版本，用于将模块特定的数据版本持久化到RDB文件，以便模块能够从RDB文件加载旧表示。

虽然一开始处理RDB加载、保存和AOF重写可能看起来很复杂，但模块API提供了非常高级的函数来处理所有这些，而不需要用户处理读/写错误，因此从实际角度来看，为Redis编写一个新的数据结构是一个简单的任务。

非常容易理解但完整的本地类型实现示例可在Redis分发中的`/modules/hellotype.c`文件中找到。
鼓励读者通过查看这个示例实现来阅读文档，以了解如何在实践中应用这些东西。

注册新数据类型
===

为了将新的本机类型注册到Redis核心中，模块需要声明一个全局变量来保存对数据类型的引用。
注册数据类型的API将返回一个数据类型引用，该引用将存储在全局变量中。

    static RedisModuleType *MyType;
    #define MYTYPE_ENCODING_VERSION 0

    int RedisModule_OnLoad(RedisModuleCtx *ctx) {
	RedisModuleTypeMethods tm = {
	    .version = REDISMODULE_TYPE_METHOD_VERSION,
	    .rdb_load = MyTypeRDBLoad,
	    .rdb_save = MyTypeRDBSave,
	    .aof_rewrite = MyTypeAOFRewrite,
	    .free = MyTypeFree
	};

        MyType = RedisModule_CreateDataType(ctx, "MyType-AZ",
		MYTYPE_ENCODING_VERSION, &tm);
        if (MyType == NULL) return REDISMODULE_ERR;
    }

从上面的示例中，您可以看到只需要一个API调用来注册新类型。然而，有一些函数指针作为参数传递。某些是可选的，而某些是必需的。上述方法集合*必须*传递，而`.digest`和`.mem_usage`是可选的，目前在模块内部实际上不支持它们，所以现在您可以忽略它们。

`ctx`参数是在`OnLoad`函数中接收到的上下文。
类型`name`是一个9个字符的名称，在字符集中包括
从大写字母`A-Z`，小写字母`a-z`，数字`0-9`，以及下划线`_`和减号`-`字符。

请注意，**此名称在Redis生态系统中的每个数据类型中必须是唯一的**，所以要有创意，如果有意义的话，可以同时使用小写和大写字母，并尝试在类型名称中混合使用模块作者的名称，以创建一个9个字符的唯一名称。

**注意：**非常重要的是，名称必须确切地为9个字符，否则类型的注册将失败。阅读更多以了解原因。

例如，如果我正在构建*b-tree*数据结构，并且我的名字是*antirez*，我会将自己的类型命名为**btree1-az**。将该名称转换为64位整数，并在保存该类型时将其存储在RDB文件中，在加载RDB数据时将用于解析能够加载数据的模块。如果Redis找不到相匹配的模块，则将整数转换回名称，以提供一些线索给用户，指示缺少哪个模块以加载数据。

类型名称也用作在调用具有注册类型的键时“TYPE”命令的回复。

`encver`参数是模块用来存储数据的编码版本，储存在RDB文件中。例如，我可以从版本0开始，但是当我发布2.0版本的模块时，我可以切换到更好的编码方式。新模块将注册为编码版本1，因此保存新的RDB文件时，新版本将被存储在磁盘上。然而，在加载RDB文件时，即使有不同编码版本的数据被找到（并且编码版本作为参数传递给`rdb_load`方法），模块的`rdb_load`方法仍将被调用，以便模块仍然能够加载旧的RDB文件。

最后一个参数是一个结构体，用于将类型方法传递给注册函数：`rdb_load`、`rdb_save`、`aof_rewrite`、`digest`、`free`和`mem_usage`都是具有以下原型和用途的回调函数：

    typedef void *(*RedisModuleTypeLoadFunc)(RedisModuleIO *rdb, int encver);
    typedef void (*RedisModuleTypeSaveFunc)(RedisModuleIO *rdb, void *value);
    typedef void (*RedisModuleTypeRewriteFunc)(RedisModuleIO *aof, RedisModuleString *key, void *value);
    typedef size_t (*RedisModuleTypeMemUsageFunc)(void *value);
    typedef void (*RedisModuleTypeDigestFunc)(RedisModuleDigest *digest, void *value);
    typedef void (*RedisModuleTypeFreeFunc)(void *value);

* `rdb_load` 在从 RDB 文件加载数据时被调用。它以与 `rdb_save` 生成的相同格式加载数据。
* `rdb_save` 在将数据保存到 RDB 文件时被调用。
* `aof_rewrite` 在重写 AOF 文件时被调用，模块需要告诉 Redis 重新创建给定键内容的命令序列是什么。
* `digest` 在执行 `DEBUG DIGEST` 并找到包含该模块类型的键时被调用。目前尚未实现该功能，因此可以将该函数留空。
* `mem_usage` 在 `MEMORY` 命令要求获取特定键消耗的总内存时被调用，用于获取模块值使用的字节数。
* `free` 在通过 `DEL` 或其他方式删除具有模块本地类型的键时被调用，以便让模块回收与该值关联的内存。

好的，但是为什么模块类型需要一个9个字符的名称？

哦，我明白你需要理解这个，所以这里有一个非常具体的
解释。

当Redis持久化到RDB文件时，特定的模块数据类型也需要进行持久化。现在RDB文件是一系列键值对，格式如下：

    [1 byte type] [key] [a type specific value]

1 字节类型用于标识字符串、列表、集合等。对于模块数据，它被设置为特殊值 `module data`，但是当然这还不够，我们需要相应的信息来将特定的值与能够加载并处理它的特定模块类型关联起来。

所以当我们保存关于模块的“类型特定值”时，我们使用一个64位整数作为前缀。 64位足够大，可以存储查找能够处理该特定类型的模块所需的信息，但是又足够短，可以在RDB中为存储的每个模块值添加前缀，而不会使最终的RDB文件太大。 同时，使用64位“签名”作为值的前缀的解决方案不需要在RDB头中定义特定类型的模块列表之类的奇怪事情。 一切都非常简单。

因此，为了可靠地标识给定模块，您可以在64位中存储什么？如果您构建一个由64个符号组成的字符集，您可以轻松地存储6位的9个字符，并且您还剩下10位，用于存储类型的"编码版本"，以便相同的类型可以在未来发展并为RDB文件提供不同、更高效或更新的序列化格式。

所以每个模块值之前存储的64位前缀如下：

    6|6|6|6|6|6|6|6|6|10

第一个 9 个元素是 6 位字符，最后的 10 位是编码版本。

当加载回RDB文件时，它读取64位值，掩码最后10位，并在模块类型缓存中搜索匹配的模块。
当找到匹配的模块时，调用加载RDB文件值的方法，并将10位编码版本作为参数传递给模块，这样模块就知道要加载哪个版本的数据布局，如果它支持多个版本。

现在有趣的是，如果无法解析模块类型，因为没有加载具有此签名的模块，
我们可以将64位值转换回9个字符的名称，并向用户打印包含模块类型名称的错误！
这样他或她立即意识到问题所在。

设置和获取键值对
---

在注册我们的新数据类型时，需要在`RedisModule_OnLoad()`函数中进行，然后我们还需要能够设置Redis键并将我们的本地类型作为值。

这通常发生在写入数据到键的命令的上下文中。
本机类型的 API 允许设置和获取键到模块本机数据类型，并测试给定的键是否已经关联到特定数据类型的值。

该 API 使用常规模块 `RedisModule_OpenKey()` 低级键访问接口来处理此问题。以下是将本地类型的私有数据结构设置为 Redis 键的示例:

    RedisModuleKey *key = RedisModule_OpenKey(ctx,keyname,REDISMODULE_WRITE);
    struct some_private_struct *data = createMyDataStructure();
    RedisModule_ModuleTypeSetValue(key,MyType,data);

使用`RedisModule_ModuleTypeSetValue()`函数时，需要打开一个用于写入的键句柄，并传入三个参数：键句柄、原生类型的引用（在类型注册时获取），以及包含实现模块原生类型的私有数据的`void*`指针。

请注意，Redis对您的数据内容一无所知。它只会调用您在方法注册时提供的回调函数，以执行类型上的操作。

同样，我们可以使用此函数从密钥中检索私有数据：

    struct some_private_struct *data;
    data = RedisModule_ModuleTypeGetValue(key);

我们还可以测试一个键是否具有我们的原生类型作为值：

    if (RedisModule_ModuleTypeGetType(key) == MyType) {
        /* ... do something ... */
    }

但是为了调用正确的方法，我们需要检查密钥是否为空，是否包含正确类型的值，诸如此类。因此，实现写入我们本地类型的命令的惯用代码如下所示：

    RedisModuleKey *key = RedisModule_OpenKey(ctx,argv[1],
        REDISMODULE_READ|REDISMODULE_WRITE);
    int type = RedisModule_KeyType(key);
    if (type != REDISMODULE_KEYTYPE_EMPTY &&
        RedisModule_ModuleTypeGetType(key) != MyType)
    {
        return RedisModule_ReplyWithError(ctx,REDISMODULE_ERRORMSG_WRONGTYPE);
    }

如果我们成功验证了键的类型不错误，并且我们要写入它，通常情况下如果键是空的，我们希望创建一个新的数据结构，或者如果已经有一个与该键关联的值，我们希望检索到该值的引用。

    /* Create an empty value object if the key is currently empty. */
    struct some_private_struct *data;
    if (type == REDISMODULE_KEYTYPE_EMPTY) {
        data = createMyDataStructure();
        RedisModule_ModuleTypeSetValue(key,MyTyke,data);
    } else {
        data = RedisModule_ModuleTypeGetValue(key);
    }
    /* Do something with 'data'... */

免费方法
---

如前所述，当Redis需要释放保存原生类型值的键时，它需要模块的帮助来释放内存。这就是为什么我们在类型注册过程中传递了一个`free`回调函数的原因：

    typedef void (*RedisModuleTypeFreeFunc)(void *value);

以下是一个简单的实现 free 方法的示例，
假设我们的数据结构是由单个分配组成的：

    void MyTypeFreeCallback(void *value) {
        RedisModule_Free(value);
    }

然而，一个更真实的例子将调用一些函数来执行更复杂的内存回收，通过将void指针转换为某种结构并释放组成值的所有资源。

RDB加载和保存方法
---

`RDB` 保存和加载回调需要在磁盘上创建（和加载回来）数据类型的表示。Redis 提供了高级 API，可以自动将以下类型的数据存储在 RDB 文件中：

* 无符号64位整数。
* 有符号64位整数。
* 浮点数。
* 字符串。

模块可以使用上述基本类型找到一种可行的表示方法。但请注意，虽然整数和双精度值以一种与体系结构和字节顺序无关的方式进行存储和加载，但如果你使用原始字符串保存API，例如将结构保存在磁盘上，你必须自己处理这些细节。

这是执行RDB保存和加载的函数列表：

    void RedisModule_SaveUnsigned(RedisModuleIO *io, uint64_t value);
    uint64_t RedisModule_LoadUnsigned(RedisModuleIO *io);
    void RedisModule_SaveSigned(RedisModuleIO *io, int64_t value);
    int64_t RedisModule_LoadSigned(RedisModuleIO *io);
    void RedisModule_SaveString(RedisModuleIO *io, RedisModuleString *s);
    void RedisModule_SaveStringBuffer(RedisModuleIO *io, const char *str, size_t len);
    RedisModuleString *RedisModule_LoadString(RedisModuleIO *io);
    char *RedisModule_LoadStringBuffer(RedisModuleIO *io, size_t *lenptr);
    void RedisModule_SaveDouble(RedisModuleIO *io, double value);
    double RedisModule_LoadDouble(RedisModuleIO *io);

该模块不需要进行任何错误检查的功能，可以始终假设调用成功。

**作为一个例子，想象我有一个实现了双精度值数组的本地类型，具有以下结构：**

    struct double_array {
        size_t count;
        double *values;
    };

我的`rdb_save`方法可能如下所示：

    void DoubleArrayRDBSave(RedisModuleIO *io, void *ptr) {
        struct dobule_array *da = ptr;
        RedisModule_SaveUnsigned(io,da->count);
        for (size_t j = 0; j < da->count; j++)
            RedisModule_SaveDouble(io,da->values[j]);
    }

我们所做的就是存储每个双精度值后面的元素数量。因此，当稍后需要在`rdb_load`方法中加载这个结构时，我们将执行以下操作：

    void *DoubleArrayRDBLoad(RedisModuleIO *io, int encver) {
        if (encver != DOUBLE_ARRAY_ENC_VER) {
            /* We should actually log an error here, or try to implement
               the ability to load older versions of our data structure. */
            return NULL;
        }

        struct double_array *da;
        da = RedisModule_Alloc(sizeof(*da));
        da->count = RedisModule_LoadUnsigned(io);
        da->values = RedisModule_Alloc(da->count * sizeof(double));
        for (size_t j = 0; j < da->count; j++)
            da->values[j] = RedisModule_LoadDouble(io);
        return da;
    }

加载回调函数只是从我们在RDB文件中存储的数据中重新构建出数据结构。

请注意，虽然写入和读取磁盘的API没有错误处理，但是在加载回调函数中，如果读取的内容看起来不正确，回调函数可能会返回NULL表示错误。在这种情况下，Redis将会发生严重错误。

AOF重写

    void RedisModule_EmitAOF(RedisModuleIO *io, const char *cmdname, const char *fmt, ...);

处理多种编码
---

    WORK IN PROGRESS

分配内存
---

模块的数据类型应尽量使用`RedisModule_Alloc()`函数族来分配、重新分配和释放用于实现本地数据结构的堆内存（详见其他Redis模块文档的详细信息）。

这不仅仅对于Redis能够记录模块使用的内存是有用的，而且还有更多的优势：

* Redis使用`jemalloc`分配器，通常可以避免使用libc分配器可能导致的碎片问题。
* 在从RDB文件加载字符串时，原生类型API能够直接返回通过`RedisModule_Alloc()`分配的字符串，这样模块就可以直接将这段内存链接到数据结构表示中，避免了不必要的数据复制。

即使您使用实施数据结构的外部库，模块API提供的分配函数与`malloc()`、`realloc()`、`free()`和`strdup()`完全兼容，因此转换库以使用这些函数应该是微不足道的。

如果您有一个使用libc `malloc()`的外部库，并且想避免手动替换所有调用为Redis模块API调用，可以使用简单的宏来替换libc调用为Redis API调用。类似这样的方法可能有效：

    #define malloc RedisModule_Alloc
    #realloc为RedisModule_Realloc
    #define free RedisModule_Free
    #define strdup RedisModule_Strdup

然而，请记住混合使用libc调用和Redis API调用会导致问题和崩溃，所以如果您使用宏替换调用，您需要确保所有调用都被正确替换，并且使用替换后的调用的代码不会尝试使用使用libc `malloc()`分配的指针来调用`RedisModule_Free()`。
