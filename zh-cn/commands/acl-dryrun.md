用给定的用户模拟执行给定的命令。
这个命令可以用来测试给定用户的权限，而不必启用该用户或引起运行命令的副作用。

@examples

```
> ACL SETUSER VIRGINIA +SET ~*
"OK"
> ACL DRYRUN VIRGINIA SET foo bar
"OK"
> ACL DRYRUN VIRGINIA GET foo bar
"This user has no permissions to run the 'GET' command"
```
