`LATENCY DOCTOR`命令报告与延迟相关的不同问题，并提供可能的解决方法建议。

这个命令是延迟监控框架中最强大的分析工具，并能够提供额外的统计数据，如延迟峰值之间的平均间隔时间、中位差异以及对事件的人类可读分析。对于某些事件，例如`fork`，还提供了额外的信息，如系统创建进程的速率。

以下是您应该在Redis邮件列表中发布的输出，如果您正在寻求有关延迟相关问题的帮助，请贴上它。

@examples

```
127.0.0.1:6379> latency doctor

Dave, I have observed latency spikes in this Redis instance.
You don't mind talking about it, do you Dave?

1. command: 5 latency spikes (average 300ms, mean deviation 120ms,
    period 73.40 sec). Worst all time event 500ms.

I have a few advices for you:

- Your current Slow Log configuration only logs events that are
    slower than your configured latency monitor threshold. Please
    use 'CONFIG SET slowlog-log-slower-than 1000'.
- Check your Slow Log to understand what are the commands you are
    running which are too slow to execute. Please check
    http://redis.io/commands/slowlog for more information.
- Deleting, expiring or evicting (because of maxmemory policy)
    large objects is a blocking operation. If you have very large
    objects that are often deleted, expired, or evicted, try to
    fragment those objects into multiple smaller objects.
```

**注意：**医生的心理行为不稳定，因此我们建议小心与其交互。

有关更多信息，请参阅[延迟监控框架页面][lm]。

[lm]：/topics/latency-monitor
