---
title: "字符串内部详细信息"
linkTitle: "字符串内部"
weight: 1
description: Redis字符串的原始实现指南
aliases:
  - /topics/internals-sds
---

**注意：这个文档是Redis的创始人Salvatore Sanfilippo在Redis开发早期（约2010年）撰写的。虚拟内存自Redis 2.6版本起已经被弃用，因此这份文档只是出于历史兴趣而保留。**

Redis字符串的实现包含在`sds.c`文件中（`sds`表示简单动态字符串）。该实现可作为独立库使用，位于[https://github.com/antirez/sds](https://github.com/antirez/sds)。

`sds.h`中声明的C结构`sdshdr`代表一个Redis字符串：

    struct sdshdr {
        long len;
        long free;
        char buf[];
    };

`buf` 字符数组存储实际的字符串。

`len`字段存储了`buf`的长度。这使得获取Redis字符串的长度成为了O(1)操作。

`free`字段存储可供使用的额外字节数。

一起，`len`和`free`字段可以被视为保存`buf`字符数组的元数据。

创建 Redis 字符串
---


一个名为`sds`的新数据类型在`sds.h`中被定义为一个字符指针的同义词：

    typedef char *sds;

在`sds.c`中定义的`sdsnewlen`函数创建一个新的Redis字符串：

    sds sdsnewlen(const void *init, size_t initlen) {
        struct sdshdr *sh;

        sh = zmalloc(sizeof(struct sdshdr)+initlen+1);
    #ifdef SDS_ABORT_ON_OOM
        if (sh == NULL) sdsOomAbort();
    #else
        if (sh == NULL) return NULL;
    #endif
        sh->len = initlen;
        sh->free = 0;
        if (initlen) {
            if (init) memcpy(sh->buf, init, initlen);
            else memset(sh->buf,0,initlen);
        }
        sh->buf[initlen] = '\0';
        return (char*)sh->buf;
    }

记住，Redis字符串是类型为`struct sdshdr`的变量。但是`sdsnewlen`返回一个字符指针！！

这是一个技巧，需要一些解释。

假设我使用`sdsnewlen`创建了一个Redis字符串，如下所示：

    sdsnewlen("redis", 5);

这将创建一个新的变量，类型为`struct sdshdr`，为`len`和`free`字段分配内存，以及为`buf`字符数组分配内存。

    sh = zmalloc(sizeof(struct sdshdr)+initlen+1); // initlen is length of init argument.

在`sdlenwlen`成功创建Redis字符串后，结果类似于：

    -----------
    |5|0|redis|
    -----------
    ^   ^
    sh  sh->buf

`sdsnewlen`将`sh->buf`返回给调用者。

如果您需要释放由`sh`指向的Redis字符串，您会怎么做？

你想要指针`sh`，但你只有指针`sh->buf`。

你能从`sh->buf`得到指针`sh`吗？

是的。指针运算。请注意上面的ASCII图形。如果您从`sh->buf`中减去两个`long`的大小，您就会得到指针`sh`。

`sizeof` 两个长整型的大小恰好是 `struct sdshdr` 的大小。

看` sdslen`函数，看看这个巧妙的技巧起作用：

    size_t sdslen(const sds s) {
        struct sdshdr *sh = (void*) (s-(sizeof(struct sdshdr)));
        return sh->len;
    }

了解了这个技巧，您可以轻松地浏览`sds.c`中的其他函数。

Redis字符串的实现被隐藏在只接受字符指针的接口后面。Redis字符串的用户不需要关心它是如何实现的，可以将Redis字符串视为字符指针。
