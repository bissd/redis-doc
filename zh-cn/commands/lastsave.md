以成功执行的最后一次DB保存的UNIX时间返回。
客户端可以通过阅读`LASTSAVE`值来检查`BGSAVE`命令是否成功，
然后发出`BGSAVE`命令，并在每隔N秒的定期间隔内检查`LASTSAVE`是否更改。Redis在启动时认为数据库已成功保存。
