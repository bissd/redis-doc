杀死当前正在执行的函数。


`FUNCTION KILL` 命令仅可用于在执行过程中未修改数据集的函数（因为停止只读函数不会违反脚本引擎的原子性保证）。

请参考[Redis函数介绍](/topics/functions-intro)以获取更多信息。
