该命令显示最近的ACL安全事件列表：

1. 由于在`AUTH`或`HELLO`中无法通过身份验证，连接失败。
2. 因违反当前ACL规则，命令被拒绝。
3. 因无法访问当前ACL规则中不允许的键，命令被拒绝。

可选参数指定要显示的条目数。默认情况下，最多返回十个失败记录。特殊的 `RESET` 参数可清除日志。条目从最近开始显示。

@examples

```
> AUTH someuser wrongpassword
(error) WRONGPASS invalid username-password pair
> ACL LOG 1
1)  1) "count"
    2) (integer) 1
    3) "reason"
    4) "auth"
    5) "context"
    6) "toplevel"
    7) "object"
    8) "AUTH"
    9) "username"
   10) "someuser"
   11) "age-seconds"
   12) "8.038"
   13) "client-info"
   14) "id=3 addr=127.0.0.1:57275 laddr=127.0.0.1:6379 fd=8 name= age=16 idle=0 flags=N db=0 sub=0 psub=0 ssub=0 multi=-1 qbuf=48 qbuf-free=16842 argv-mem=25 multi-mem=0 rbs=1024 rbp=0 obl=0 oll=0 omem=0 tot-mem=18737 events=r cmd=auth user=default redir=-1 resp=2"
   15) "entry-id"
   16) (integer) 0
   17) "timestamp-created"
   18) (integer) 1675361492408
   19) "timestamp-last-updated"
   20) (integer) 1675361492408
```

每个日志条目由以下字段组成：

1. `count`：在60秒内检测到的安全事件数量。
2. `reason`：记录安全事件的原因。可以是`command`（命令）、`key`（密钥）、`channel`（通道）或`auth`（认证）。
3. `context`：检测到安全事件的上下文。可以是`toplevel`（顶层）、`multi`（多重）、`lua`（Lua脚本）或`module`（模块）。
4. `object`：用户无权访问的资源。当原因为`auth`时，为`auth`。
5. `username`：执行导致安全事件的命令的用户名，或者失败的认证尝试的用户名。
6. `age-seconds`：日志条目的年龄（以秒为单位）。
7. `client-info`：显示导致安全事件的客户端的客户端信息。
8. `entry-id`：自服务器进程启动以来的条目序列号（从0开始）。也可用于检查是否“丢失”了条目，即它们是否处于周期之间。
9. `timestamp-created`：条目被首次创建时的UNIX时间戳（以毫秒为单位）。
10. `timestamp-last-updated`：条目最后更新时的UNIX时间戳（以毫秒为单位）。
