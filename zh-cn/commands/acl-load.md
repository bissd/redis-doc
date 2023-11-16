当Redis配置为使用ACL文件（使用`aclfile`配置选项）时，此命令将从文件中重新加载ACL，用文件中定义的ACL规则替换所有当前的ACL规则。此命令确保具有*全有或全无*的行为，即：

* 如果文件中的每行都有效，则加载所有访问控制列表（ACL）。
* 如果文件中的一行或多行无效，则不加载任何内容，并继续使用服务器内存中定义的旧ACL规则。

@examples

```
> ACL LOAD
+OK

> ACL LOAD
-ERR /tmp/foo:1: Unknown command or category name in ACL...
```
