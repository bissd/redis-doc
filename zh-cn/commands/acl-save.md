在Redis配置为使用ACL文件（使用`aclfile`配置选项）时，此命令将当前定义的ACL从服务器内存保存到ACL文件中。

@examples

```
> ACL SAVE
+OK

> ACL SAVE
-ERR There was an error trying to save the ACLs. Please check the server logs for more information
```
