# go-mmproxy

这是 [mmproxy](https://github.com/cloudflare/mmproxy) 的 Go 语言重新实现，旨在提高 mmproxy 的运行时稳定性，同时在连接和数据包吞吐量方面提供潜在的更高性能。

`go-mmproxy` 是一个独立的应用程序，它解包 HAProxy 的 [PROXY 协议](http://www.haproxy.org/download/1.8/doc/proxy-protocol.txt)（也被 NGINX 等其他项目采用），以便与最终服务器的网络连接来自客户端（而不是代理服务器）的 IP 地址和端口号。
由于它们共享基本机制，[Cloudflare 关于 mmproxy 的博客文章](https://blog.cloudflare.com/mmproxy-creative-way-of-preserving-client-ips-in-spectrum/) 很好地阐述了 `go-mmproxy` 的底层工作原理。

## 构建

```shell
go install github.com/path-network/go-mmproxy@latest
```

您需要至少 `go 1.21` 才能构建 `go-mmproxy` 二进制文件。
如果您的包管理器没有足够新的 golang 版本，请参阅 [Go 的入门指南](https://golang.org/doc/install)。

## 要求

`go-mmproxy` 必须运行在：

- 与代理目标相同的服务器上，因为通信通过环回接口进行；
- 作为 root 用户或具有 `CAP_NET_ADMIN` 能力，以便能够设置 `IP_TRANSPARENT` 套接字选项。

## 运行

### 路由设置

将所有源自环回的流量路由回环回：

```shell
ip rule add from 127.0.0.1/8 iif lo table 123
ip route add local 0.0.0.0/0 dev lo table 123

ip -6 rule add from ::1/128 iif lo table 123
ip -6 route add local ::/0 dev lo table 123
```

如果 `--mark` 选项提供给 `go-mmproxy`，所有路由到环回接口的数据包都将设置标记。
这可以用于设置更高级的 iptables 路由规则，例如当您需要将来自环回的流量路由到机器外部时。

#### 路由 UDP 数据包

由于 UDP 是无连接的，如果套接字绑定到 `0.0.0.0`，内核堆栈将搜索接口以向欺骗的源地址发送回复——而不是仅仅使用它接收原始数据包的接口。
找到的接口很可能不是环回接口，这将避免上述规则。
最简单的解决方法是将最终服务器的监听器绑定到 `127.0.0.1`（或 `::1`）。
通常也建议这样做，以避免接收非代理连接。

### 启动 go-mmproxy

```
Usage of ./go-mmproxy:
  -4 string
    	IPv4 流量将转发到的地址 (默认 "127.0.0.1:443")
  -6 string
    	IPv6 流量将转发到的地址 (默认 "[::1]:443")
  -allowed-subnets string
    	包含代理服务器允许子网的文件路径
  -close-after int
    	UDP 套接字将被清理的秒数 (默认 60)
  -l string
    	代理监听的地址 (默认 "0.0.0.0:8443")
  -listeners int
    	监听地址将打开的监听套接字数量 (Linux 3.9+) (默认 1)
  -mark int
    	出站数据包将设置的标记
  -p string
    	将代理的协议: tcp, udp (默认 "tcp")
  -v int
    	0 - 不记录单个连接
    	1 - 记录单个连接中发生的错误
    	2 - 记录单个连接的所有状态更改
```

示例调用：

```shell
sudo ./go-mmproxy -l 0.0.0.0:25577 -4 127.0.0.1:25578 -6 [::1]:25578 --allowed-subnets ./path-prefixes.txt
```

## 基准测试

### 设置

基准测试在 Dell XPS 9570 上运行，配备 Intel Core i9-8950HK CPU @ 2.90GHz（12 个逻辑核心）。代理发送流量的上游服务由 [bpf-echo](https://github.com/path-network/bpf-echo) 服务器模拟。
流量由 [tcpkali](https://github.com/satori-com/tcpkali) v1.1.1 生成。

在所有情况下，负载生成都使用以下命令（50 个连接，10 秒运行时，每个连接发送 PROXYv1 头部，使用 `PING\r\n` 作为 TCP 消息）：

```
tcpkali -c 50 -T 10s -e1 'PROXY TCP4 127.0.0.1 127.0.0.1 \{connection.uid} 25578\r\n' -m 'PING\r\n' 127.0.0.1:1122
```

### 结果

|                         | ⇅ Mbps    | ↓ Mbps    | ↑ Mbps    | ↓ pkt/s   | ↑ pkt/s   |
| ----------------------- | --------- | --------- | --------- | --------- | --------- |
| cloudflare/mmproxy      | 1524.454  | 756.385   | 768.069   | 70365.9   | 65921.9   |
| go-mmproxy GOMAXPROCS=1 | 7418.312  | 2858.794  | 4559.518  | 262062.7  | 391334.6  |
| go-mmproxy              | 45483.233 | 16142.348 | 29340.885 | 1477889.6 | 2518271.5 |
| no proxy                | 52640.116 | 22561.129 | 30078.987 | 2065805.4 | 2581621.3 |

![结果条形图](benchmark.png)


