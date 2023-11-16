这是 `EVAL` 命令的只读变体，不能执行修改数据的命令。

请参阅关于何时使用此命令与`EVAL`的更多信息，请参见[只读脚本](/docs/manual/programmability/#read-only-scripts)。

有关`EVAL`脚本的更多信息，请参阅[Eval脚本简介](/topics/eval-intro)。

@examples

```
> SET mykey "Hello"
OK

> EVAL_RO "return redis.call('GET', KEYS[1])" 1 mykey
"Hello"

> EVAL_RO "return redis.call('DEL', KEYS[1])" 1 mykey
(error) ERR Error running script (call to b0d697da25b13e49157b2c214a4033546aba2104): @user_script:1: @user_script: 1: Write commands are not allowed from read-only scripts.
```
