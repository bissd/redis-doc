`PEXPIRETIME`具有与`EXPIRETIME`相同的语义，但返回的是绝对的Unix过期时间戳（以毫秒为单位），而不是以秒为单位。

@examples

```cli
SET mykey "Hello"
PEXPIREAT mykey 33177117420000
PEXPIRETIME mykey
```
