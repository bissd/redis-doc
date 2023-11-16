这个命令控制在下一个由连接执行的命令中跟踪键的情况，当在“OPTIN”或“OPTOUT”模式下启用跟踪时。
有关背景信息，请参阅[客户端缓存文档](/topics/client-side-caching)。

当启用Redis跟踪时，可以通过`CLIENT TRACKING`命令指定`OPTIN`或`OPTOUT`选项，以使只读命令中的键不会自动被服务器记住以供以后失效。当我们处于`OPTIN`模式时，可以在下一条命令之前调用`CLIENT CACHING yes`来启用键的跟踪。同样，当我们处于`OPTOUT`模式且键通常被跟踪时，可以使用`CLIENT CACHING no`来避免在下一条命令中跟踪键。

基本上，该命令在连接中设置一个状态，该状态仅对下一次命令执行有效，该命令将修改客户端跟踪的行为。
