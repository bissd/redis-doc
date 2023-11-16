`BITFIELD` 命令的只读变体。
和原始的 `BITFIELD` 类似，但只接受 `!GET` 子命令，并且可以安全地在只读副本中使用。

由于原始的`BITFIELD`命令具有`!SET`和`!INCRBY`选项，因此在Redis命令表中技术上标记为写入命令。
因此，在Redis Cluster中的只读复制会将其重定向到主实例，即使连接处于只读模式（请参阅Redis Cluster的`READONLY`命令）。

自 Redis 6.2 起，引入了 `BITFIELD_RO` 变体，以允许在只读副本中进行 `BITFIELD` 行为，而不会破坏命令标志的兼容性。

有关更多详细信息，请参阅原始的`BITFIELD`。

@examples

```
BITFIELD_RO hello GET i8 16
```
