返回有关函数和库的信息。

您可以使用可选的`LIBRARYNAME`参数来指定匹配库名称的模式。
可选的`WITHCODE`修改器将导致服务器在回复中包含库的源代码实现。

响应中为每个库提供了以下信息：

* **library_name：**库的名称。
* **engine：**库的引擎。
* **functions：**库中包含的函数列表。
  每个函数具有以下字段：
  * **name：**函数的名称。
  * **description：**函数的描述。
  * **flags：**包含[函数标识](/docs/manual/programmability/functions-intro/#function-flags)的数组。
* **library_code：**库的源代码（当给出`WITHCODE`修饰符时）。

更多信息请参阅[Redis函数介绍](/topics/functions-intro)。
