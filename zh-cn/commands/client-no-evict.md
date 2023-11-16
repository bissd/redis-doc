`CLIENT NO-EVICT`命令为当前连接设置[客户端驱逐](/topics/clients#client-eviction)模式。

如果打开了客户端驱逐功能并进行了配置，即使超过了配置的客户端驱逐阈值，当前连接也将被排除在客户端驱逐过程之外。

当关闭时，当前客户端将重新包含在潜在待清退客户端池中（如果需要的话清退）。

请参阅[客户端剔除](/topics/clients#client-eviction)以获取更多详细信息。
