加载一个库到Redis中。

这个命令接收一个必需的参数，这个参数是实现该库的源代码。
该库的有效负载必须以Shebang语句开头，该语句提供了关于库的元数据（如使用的引擎和库的名称）。
Shebang格式：`#!<引擎名称> 名称=<库名称>`。目前引擎名称必须为`lua`。

对于Lua引擎，应使用[`redis.register_function()` API](/topics/lua-api#redis.register_function)声明一个或多个库的入口点。
加载后，您可以使用`FCALL`命令（或在适用时使用`FCALL_RO`命令）调用库中的函数。

在尝试加载一个已经存在的库时，Redis服务器会返回错误。
`REPLACE` 修饰符可以改变这种行为，并用新内容覆盖现有的库。

命令在以下情况下会返回错误：

* 提供了无效的_engine-name_。
* 图书馆的名称已经存在，没有使用 `REPLACE` 修饰符。
* 在库中创建了一个名称已在另一个库中存在的函数（即使指定了 `REPLACE`）。
* 引擎在创建库的函数时失败（例如由于编译错误）。
* 图书馆没有声明任何函数。

有关更多信息，请参阅[Redis函数介绍](/topics/functions-intro)。

@examples

以下示例将创建一个名为 `mylib` 的库，其中包含一个名为 `myfunc` 的函数，该函数返回它接收到的第一个参数。

```
redis> FUNCTION LOAD "#!lua name=mylib \n redis.register_function('myfunc', function(keys, args) return args[1] end)"
mylib
redis> FCALL myfunc 0 hello
"hello"
```
