---
title: "Go 指南"
linkTitle: "Go"
description: 将您的Go应用程序连接到Redis数据库
weight: 2
aliases:
  - /docs/clients/go/
---

安装 Redis 和 Redis 客户端，然后将您的 Go 应用程序连接到 Redis 数据库。

## go-redis

go-redis是一个用于Go语言的Redis客户端库。它提供了简单、易于使用的API来与Redis数据库进行交互。它支持所有的Redis命令和管道操作，并且还提供了一系列的辅助函数来简化常见的操作。go-redis还支持分布式锁、发布/订阅、事务等高级功能。它是一个功能强大、稳定可靠的Redis客户端库，适用于各种规模的应用程序。

[`go-redis`](https://github.com/redis/go-redis) 为各种 Redis 版本提供了 Go 客户端，并为每个 Redis 命令提供了类型安全的 API。

### 安装

`go-redis`支持最后两个Go版本，并且仅适用于Go模块。因此，首先，您需要初始化一个Go模块：

```
go mod init github.com/my/repo
```

To install go-redis/v9:

```
go get github.com/redis/go-redis/v9
```

### 连接

要连接到Redis服务器，请按照以下步骤操作：

```go
import (
	"context"
	"fmt"
	"github.com/redis/go-redis/v9"
)

func main() {
    client := redis.NewClient(&redis.Options{
        Addr:	  "localhost:6379",
        Password: "", // no password set
        DB:		  0,  // use default DB
    })
}
```

另一种连接方式是使用连接字符串。

```go
opt, err := redis.ParseURL("redis://<user>:<pass>@localhost:6379/<db>")
if err != nil {
	panic(err)
}

client := redis.NewClient(opt)
```

存储和检索一个简单的字符串。

```go
ctx := context.Background()

err := client.Set(ctx, "foo", "bar", 0).Err()
if err != nil {
    panic(err)
}

val, err := client.Get(ctx, "foo").Result()
if err != nil {
    panic(err)
}
fmt.Println("foo", val)
```

存储和检索地图。

```go
session := map[string]string{"name": "John", "surname": "Smith", "company": "Redis", "age": "29"}
for k, v := range session {
    err := client.HSet(ctx, "user-session:123", k, v).Err()
    if err != nil {
        panic(err)
    }
}

userSession := client.HGetAll(ctx, "user-session:123").Val()
fmt.Println(userSession)
 ```

#### 连接到Redis集群

要连接到Redis集群，请使用`NewClusterClient`。

```go
client := redis.NewClusterClient(&redis.ClusterOptions{
    Addrs: []string{":16379", ":16380", ":16381", ":16382", ":16383", ":16384"},

    // To route commands by latency or randomly, enable one of the following.
    //RouteByLatency: true,
    //RouteRandomly: true,
})
```

#### 使用TLS连接到你的生产Redis

在部署应用程序时，请使用TLS并遵循[Redis安全](/docs/management/security/)的准则。

使用此代码段与您的Redis数据库建立安全连接。

```go
// Load client cert
cert, err := tls.LoadX509KeyPair("redis_user.crt", "redis_user_private.key")
if err != nil {
    log.Fatal(err)
}

// Load CA cert
caCert, err := os.ReadFile("redis_ca.pem")
if err != nil {
    log.Fatal(err)
}
caCertPool := x509.NewCertPool()
caCertPool.AppendCertsFromPEM(caCert)

client := redis.NewClient(&redis.Options{
    Addr:     "my-redis.cloud.redislabs.com:6379",
    Username: "default", // use your Redis user. More info https://redis.io/docs/management/security/acl/
    Password: "secret", // use your Redis password
    TLSConfig: &tls.Config{
        MinVersion:   tls.VersionTLS12,
        Certificates: []tls.Certificate{cert},
        RootCAs:      caCertPool,
    },
})

//send SET command
err = client.Set(ctx, "foo", "bar", 0).Err()
if err != nil {
    panic(err)
}

//send GET command and print the value
val, err := client.Get(ctx, "foo").Result()
if err != nil {
    panic(err)
}
fmt.Println("foo", val)
```


#### 拨号 tcp: i/o 超时

当`go-redis`无法连接到Redis服务器时，会出现`dial tcp: i/o timeout`错误，例如当服务器关闭或端口受到防火墙保护。要检查Redis服务器是否在该端口上监听，请在运行`go-redis`客户端的主机上运行telnet命令。

```go
telnet localhost 6379
Trying 127.0.0.1...
telnet: Unable to connect to remote host: Connection refused
```

如果您使用Docker、Istio或任何其他服务网格/边车，请确保应用程序在容器完全可用后启动，例如，通过使用Docker进行健康检查并在Istio中使用holdApplicationUntilProxyStarts功能。
有关更多信息，请参阅[健康检查](https://docs.docker.com/engine/reference/run/#healthcheck)。

### 了解更多

* [文档](https://redis.uptrace.dev/guide/)
* [GitHub](https://github.com/redis/go-redis)
 
