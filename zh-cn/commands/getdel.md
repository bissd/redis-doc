获取“key”的值并删除该键。
该命令类似于“GET”，只是在成功时还会删除键（仅当键的值类型为字符串时）。

@examples

```cli
SET mykey "Hello"
GETDEL mykey
GET mykey
```
