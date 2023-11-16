该命令显示Redis服务器中当前活动的ACL规则。返回数组中的每一行定义了不同的用户，格式与redis.conf文件或外部ACL文件中使用的格式相同，因此如果需要的话，可以直接将ACL LIST命令返回的内容剪切并粘贴到配置文件中（但务必检查ACL SAVE）。

@examples

```
> ACL LIST
1) "user antirez on #9f86d081884c7d659a2feaa0c55ad015a3bf4f1b2b0b822cd15d6c15b0f00a08 ~objects:* &* +@all -@admin -@dangerous"
2) "user default on nopass ~* &* +@all"
```
