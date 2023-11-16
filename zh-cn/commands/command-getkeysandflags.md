返回完整Redis命令的键和它们的使用标志的@array-reply。

`COMMAND GETKEYSANDFLAGS` 是一个辅助命令，让您从完整的 Redis 命令中找到键，并指示每个键的用途。

`COMMAND` 提供了关于如何找到每个命令的键名的信息（参见 `firstkey`，[键规范](/topics/key-specs#logical-operation-flags) 和 `movablekeys`），
但在某些情况下，无法找到某些命令的键，因此必须解析整个命令以发现一些/所有的键名。
您可以使用 `COMMAND GETKEYS` 或 `COMMAND GETKEYSANDFLAGS` 直接从 Redis 解析命令中发现键名。

参考[key specifications](/topics/key-specs#logical-operation-flags) 了解关键标志的含义。

@examples

```cli
COMMAND GETKEYS MSET a b c d e f
COMMAND GETKEYS EVAL "not consulted" 3 key1 key2 key3 arg1 arg2 arg3 argN
COMMAND GETKEYSANDFLAGS LMOVE mylist1 mylist2 left left
```
