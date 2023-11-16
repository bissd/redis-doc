该命令会返回已存在的ACL用户所定义的所有规则。

具体来说，它列出了用户的ACL标志、密码哈希、命令、键模式、通道模式（自版本6.2起添加）和选择器（自版本7.0起添加）。
如果将来为用户添加更多元数据，可能还会返回其他附加信息。

命令规则始终以与“ACL SETUSER”命令相同的格式返回。
在7.0版本之前，键和频道作为模式数组返回，然而在7.0版本之后，它们也以与“ACL SETUSER”命令相同的格式返回。
注意：命令规则的描述反映了用户的有效权限，因此虽然可能与用于配置用户的规则集不完全相同，但在功能上是相同的。

选择器按照应用于用户的顺序进行列出，并包括有关命令、键模式和通道模式的信息。

@examples

```
> ACL SETUSER sample on nopass +GET allkeys &* (+SET ~key2)
"OK"
> ACL GETUSER sample
1) "flags"
2) 1) "on"
   2) "allkeys"
   3) "nopass"
3) "passwords"
4) (empty array)
5) "commands"
6) "+@all"
7) "keys"
8) "~*"
9) "channels"
10) "&*"
11) "selectors"
12) 1) 1) "commands"
       6) "+SET"
       7) "keys"
       8) "~key2"
       9) "channels"
       10) "&*"
```
