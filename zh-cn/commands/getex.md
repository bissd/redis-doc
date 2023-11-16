获取“key”的值，并可选择设置其过期时间。
`GETEX`类似于`GET`，但是它是一个带有额外选项的写入命令。

## 选项

`GETEX`命令支持一组选项来修改其行为：

* `EX` *秒* -- 设置指定的到期时间，单位为秒。
* `PX` *毫秒* -- 设置指定的到期时间，单位为毫秒。
* `EXAT` *时间戳-秒* -- 设置键将在指定的 Unix 时间到期，单位为秒。
* `PXAT` *时间戳-毫秒* -- 设置键将在指定的 Unix 时间到期，单位为毫秒。
* `PERSIST` -- 移除与键关联的生存时间。

@examples

```cli
SET mykey "Hello"
GETEX mykey
TTL mykey
GETEX mykey EX 60
TTL mykey
```
