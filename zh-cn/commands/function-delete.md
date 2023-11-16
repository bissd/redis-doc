删除一个库和其所有的函数。

这个命令将删除名为 _library-name_ 的库以及其中的所有函数。
如果该库不存在，服务器将返回错误。

有关更多信息，请参阅[Redis函数介绍](/topics/functions-intro)。

@examples

```
redis> FUNCTION LOAD Lua mylib "redis.register_function('myfunc', function(keys, args) return 'hello' end)"
OK
redis> FCALL myfunc 0
"hello"
redis> FUNCTION DELETE mylib
OK
redis> FCALL myfunc 0
(error) ERR Function not found
```
