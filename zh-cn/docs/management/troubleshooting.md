---
title: "Redis 故障排除"
linkTitle: "故障排除"
weight: 9
description: 如果Redis出现问题，请从这里开始。
aliases: [
    /topics/problems,
    /docs/manual/troubleshooting,
    /docs/manual/troubleshooting.md
]
---

这个页面试图帮助您解决Redis的问题。Redis项目的一部分是帮助遇到问题的人，因为我们不喜欢让他们独自面对困难。

* 如果您使用Redis时遇到**延迟问题**，而Redis在某种程度上似乎处于闲置状态一段时间，请阅读我们的[Redis延迟故障排除指南](/topics/latency)。
* Redis稳定版本通常非常可靠，但如果您**遇到崩溃问题**，如果提供调试信息，开发人员可以提供更多帮助。请阅读我们的[Redis调试指南](/topics/debugging)。
* 我们有很多用户在使用Redis时遇到崩溃问题，最终发现是**服务器内存损坏**。如果您的系统中Redis不稳定，请使用**redis-server --test-memory**测试您的内存。Redis内置的内存测试速度快且相对可靠，但如果可能的话，您应该重新启动服务器并使用[memtest86](http://memtest86.com)。

对于其他问题，请给[Redis Google Group](http://groups.google.com/group/redis-db)留言。我们非常乐意帮助您。

您还可以在 [Redis Discord 服务器](https://discord.gg/redis) 上找到帮助。

### Redis 3.0.x，2.8.x和2.6.x已知的重要Bug列表

要查找关键错误列表，请参考changelogs：

* [Redis 3.0 更新日志](https://raw.githubusercontent.com/redis/redis/3.0/00-RELEASENOTES).
* [Redis 2.8 更新日志](https://raw.githubusercontent.com/redis/redis/2.8/00-RELEASENOTES).
* [Redis 2.6 更新日志](https://raw.githubusercontent.com/redis/redis/2.6/00-RELEASENOTES).

在每个补丁发布中检查*升级紧急程度*级别，以更容易地发现包含重要修复的发布版本。

### 已知影响Redis的Linux相关错误清单。

* Ubuntu 10.04和10.10包含[错误](https://bugs.launchpad.net/ubuntu/+source/linux/+bug/666211)，可能会导致性能问题。不建议使用这些发行版附带的默认内核。有报告称这些错误会影响EC2实例，但也有用户提到服务器受到影响。
* 某些版本的Xen虚拟化程序报告fork()性能差。有关更多信息，请参见[延迟页面](/topics/latency)。
