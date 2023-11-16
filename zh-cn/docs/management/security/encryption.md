---
title: "TLS"
linkTitle: "TLS"
weight: 1
description: Redis TLS 支持
aliases: [
    /topics/encryption,
    /docs/manual/security/encryption,
    /docs/manual/security/encryption.md
]
---

Redis从版本6开始支持SSL/TLS，作为一个可选功能，需要在编译时启用。

## 入门指南

### 构建

为了使用TLS支持，您需要OpenSSL开发库（例如，在Debian/Ubuntu上是`libssl-dev`）。

使用以下命令构建Redis：
```
$ make
```

```sh
make BUILD_TLS=yes
```

### 测试

要使用TLS运行Redis测试套件，您需要为TCL启用TLS支持（即Debian/Ubuntu上的`tcl-tls`软件包）。

运行 `./utils/gen-test-certs.sh` 生成根CA证书和服务器证书。

2. 运行`./runtest --tls`或`./runtest-cluster --tls`以在TLS模式下运行Redis和Redis集群测试。

### 手动运行

为了以 TLS 模式手动运行 Redis 服务器（假设已调用 `gen-test-certs.sh` 以准备示例证书/密钥）：

    ./src/redis-server --tls-port 6379 --port 0 \
        --tls-cert-file ./tests/tls/redis.crt \
        --tls-key-file ./tests/tls/redis.key \
        --tls-ca-cert-file ./tests/tls/ca.crt

以`redis-cli`连接到此Redis服务器的方法为：

    ./src/redis-cli --tls \
        --cert ./tests/tls/redis.crt \
        --key ./tests/tls/redis.key \
        --cacert ./tests/tls/ca.crt

### 证书配置

为了支持TLS，Redis必须配置具有X.509证书和私钥。此外，还需要指定一个CA证书捆绑文件或路径，用作验证证书时的受信任根。为了支持基于DH的密码，还可以配置一个DH参数文件。例如：

```
tls-cert-file /path/to/redis.crt
tls-key-file /path/to/redis.key
tls-ca-cert-file /path/to/ca.crt
tls-dh-params-file /path/to/redis.dh
```

### TLS 监听端口

`tls-port`配置指令使得能够在指定端口上接受SSL/TLS连接。这是**除了**监听`port`上的TCP连接之外的功能，因此可以同时使用TLS和非TLS连接访问Redis的不同端口。

您可以指定“端口 0”来完全禁用非 TLS 端口。要在默认的 Redis 端口上仅启用 TLS，请使用：

```
port 0
tls-port 6379
```

### 客户端证书认证

默认情况下，Redis使用互相TLS，并要求客户端通过有效证书进行身份验证（根据`ca-cert-file`或`ca-cert-dir`指定的受信任的根CA）。

您可以使用 `tls-auth-clients no` 来禁用客户端认证。

### 复制 (Replication)

一个Redis主服务器以相同的方式处理连接的客户端和副本服务器，因此上述的`tls-port`和`tls-auth-clients`指令也适用于复制链接。

在复制服务器端，需要指定 `tls-replication yes` 来使用 TLS 对传出连接到主服务器进行加密。

### 集群

当使用Redis集群时，使用`tls-cluster yes`以启用集群总线和节点间连接的TLS。

### Sentinel

Sentinel从通用的Redis配置中继承其网络配置，因此上述的所有内容也适用于Sentinel。

连接到主服务器时，Sentinel将使用“tls-replication”指令来确定是否需要TLS或非TLS连接。

此外，`tls-replication`指令将确定Sentinel监听来自其他Sentinel的连接的端口是否也支持TLS。也就是说，只有在启用了`tls-replication`的情况下，才会配置Sentinel的`tls-port`。

### 附加配置

有关TLS协议版本、密码和密码套件的选择，还可以进行额外的TLS配置等。更多信息，请参阅自述文件`redis.conf`。

### 性能考虑

TLS给通信堆栈添加了一层，由于在SSL连接上读写、加密/解密和完整性检查等方面产生了一些开销。因此，使用TLS会导致Redis实例的可实现吞吐量降低（有关更多信息，请参阅此讨论）。

### 限制

I/O 线程目前不支持 TLS。
