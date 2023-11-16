从完整的Redis命令返回@array-reply的键。

`COMMAND GETKEYS` 是一个辅助命令，用于从完整的 Redis 命令中找出键名。

`COMMAND`提供了如何查找每个命令的键名的信息（参见`firstkey`，[key specifications](/topics/key-specs#logical-operation-flags)，和`movablekeys`），
但在某些情况下无法找到某些命令的键，然后必须解析整个命令以发现一些/所有键名。
您可以使用`COMMAND GETKEYS`或`COMMAND GETKEYSANDFLAGS`直接从Redis解析命令来发现键名。

@examples

```cli
COMMAND GETKEYS MSET a b c d e f
COMMAND GETKEYS EVAL "not consulted" 3 key1 key2 key3 arg1 arg2 arg3 argN
COMMAND GETKEYS SORT mylist ALPHA STORE outlist
```
