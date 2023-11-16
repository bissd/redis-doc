调用函数。

使用`FUNCTION LOAD`命令将函数加载到服务器。
第一个参数是已加载函数的名称。

第二个参数是输入键名称参数的数量，接下来是函数访问的所有键。
在Lua中，这些输入键的名称对于函数而言是作为回调函数的第一个参数的表格可用。

**重要:**
为了确保函数在独立部署和集群部署中正确执行，函数所访问的所有键名必须作为输入键参数明确提供。
函数**仅应**访问输入参数中指定的键名。
函数**绝不应**根据程序生成的名称或存储在数据库中的数据结构的内容访问键。

任何额外的输入参数**不应当**表示键值的名称。
这些是普通参数，并作为回调函数的第二个参数传递到 Lua 表中。

有关更多信息，请参阅[Redis可编程性](/topics/programmability)和[Redis函数介绍](/topics/functions-intro)页面。

@examples

下面的示例将创建一个名为 `mylib` 的库，其中包含一个名为 `myfunc` 的函数，该函数返回它接收到的第一个参数。

```
redis> FUNCTION LOAD "#!lua name=mylib \n redis.register_function('myfunc', function(keys, args) return args[1] end)"
"mylib"
redis> FCALL myfunc 0 hello
"hello"
```
