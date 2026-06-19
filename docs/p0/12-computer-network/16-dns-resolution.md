# DNS 解析流程

## 面试定位

DNS 是很多网络问题的第一跳。面试常问：输入域名后 DNS 怎么解析、本地缓存和递归解析器是什么、A 记录和 CNAME 区别、DNS 解析慢怎么排查、为什么同一个域名在不同机器解析结果不同。

核心回答：

DNS 把域名解析为 IP。客户端通常先查本地缓存和 hosts，再请求递归 DNS 服务器；递归 DNS 服务器必要时从根、顶级域、权威 DNS 逐级查询，最终返回记录。

## DNS 解析对象

常见记录类型：

| 类型 | 含义 |
| --- | --- |
| A | 域名到 IPv4 |
| AAAA | 域名到 IPv6 |
| CNAME | 域名别名 |
| MX | 邮件服务器 |
| TXT | 文本记录，常用于验证 |
| NS | 域的权威 DNS |
| SRV | 服务发现记录 |

Java 后端最常遇到：

- A。
- AAAA。
- CNAME。
- SRV，部分注册发现或中间件场景。

## 客户端解析流程

一次典型解析：

```text
应用请求 example.com
  -> 检查 JVM DNS 缓存
  -> 检查系统缓存
  -> 检查 /etc/hosts
  -> 查询本机配置的 DNS 递归解析器
  -> 得到 IP
  -> 建立 TCP 连接
```

实际顺序受操作系统、JVM、容器和配置影响。

Linux DNS 配置：

```bash
cat /etc/resolv.conf
cat /etc/hosts
```

## 递归解析流程

如果递归 DNS 没有缓存，会执行类似流程：

```text
客户端 -> 递归 DNS：example.com 是什么 IP
递归 DNS -> 根 DNS：.com 的 DNS 在哪
递归 DNS -> .com 顶级域 DNS：example.com 的权威 DNS 在哪
递归 DNS -> example.com 权威 DNS：example.com 的 A 记录是什么
递归 DNS -> 客户端：返回 IP
```

客户端一般不会直接访问根 DNS，而是交给递归解析器。

## CNAME 解析

CNAME 是别名：

```text
www.example.com CNAME app.example-cdn.com
app.example-cdn.com A 1.2.3.4
```

解析时需要继续查询别名目标。CDN、网关、云厂商负载均衡常用 CNAME。

排查时不能只看最终 IP，还要看 CNAME 链是否符合预期。

## TTL

TTL 是 DNS 记录可缓存时间。

```text
example.com 60 IN A 1.2.3.4
```

含义：解析结果可缓存 60 秒。

影响：

- TTL 低：变更生效快，但 DNS 查询压力大。
- TTL 高：缓存稳定，但切流和故障恢复慢。

注意 JVM、操作系统、DNS 递归器可能有自己的缓存策略，不一定完全等于权威 DNS TTL。

## DNS 和负载均衡

DNS 可返回多个 IP：

```text
example.com -> 203.0.113.10, 203.0.113.20
```

用于简单负载均衡或就近接入。

限制：

- 客户端缓存会影响切换速度。
- DNS 不感知单次请求状态。
- 客户端可能固定使用某个 IP。
- 故障 IP 在缓存过期前仍可能被访问。

因此核心服务通常还会配合 LB、注册中心或客户端负载均衡。

## 命令示例

查询 A 记录：

```bash
dig example.com A
nslookup example.com
```

查看 CNAME 链：

```bash
dig example.com
dig +trace example.com
```

指定 DNS 服务器：

```bash
dig @8.8.8.8 example.com
dig @114.114.114.114 example.com
```

只看简洁结果：

```bash
dig +short example.com
```

查看解析耗时：

```bash
curl -o /dev/null -s -w 'dns=%{time_namelookup} total=%{time_total}\n' https://example.com/
```

## Java DNS

Java 有自己的 DNS 缓存策略。

常见属性：

```text
networkaddress.cache.ttl
networkaddress.cache.negative.ttl
```

可能通过安全配置或 JVM 参数调整：

```bash
-Dsun.net.inetaddr.ttl=60
-Dsun.net.inetaddr.negative.ttl=10
```

注意：

- 不同 JDK 版本和安全管理配置默认值可能不同。
- 长时间缓存 DNS 可能导致域名切流不生效。
- 过短 TTL 会增加解析压力。

## 线上排查

### DNS 解析慢

检查：

- `/etc/resolv.conf` 中 nameserver 是否可达。
- DNS 服务器是否超时或丢包。
- 是否有多个 nameserver，第一个慢导致串行等待。
- 是否 IPv6 AAAA 查询慢。
- 容器 DNS，如 CoreDNS，是否异常。

命令：

```bash
dig example.com
time nslookup example.com
```

### 解析结果不一致

可能原因：

- 不同机器使用不同 DNS 服务器。
- 缓存未过期。
- 运营商或机房 DNS 策略不同。
- 灰度切流。
- split-horizon DNS，内外网返回不同结果。

排查：

```bash
dig @dns-server-1 example.com
dig @dns-server-2 example.com
cat /etc/resolv.conf
```

### 域名能解析但连接失败

DNS 只说明拿到了 IP，不代表端口可达。

继续排查：

```bash
nc -vz <ip> 443
curl -v --resolve example.com:443:<ip> https://example.com/
```

`--resolve` 可绕过 DNS，把域名绑定到指定 IP，同时保留 HTTPS SNI 和 Host。

## 常见追问

### DNS 用 TCP 还是 UDP

常规查询多用 UDP 53。响应过大、区域传送或某些场景会使用 TCP 53。现代 DNS 还可能使用 DoH、DoT。

### hosts 文件优先级一定最高吗

大多数系统解析会先查 hosts，但具体顺序受 NSS、系统配置和应用实现影响。排查时要看实际系统配置。

### DNS 解析成功后请求还会失败吗

会。DNS 只返回地址，后续还要 TCP 建连、TLS 握手、HTTP 处理。

## 易错点

- 把 DNS 当成只查一次的静态映射。
- 忽略 JVM DNS 缓存。
- 不看 CNAME 链，只看最终 IP。
- 解析域名后直接用 IP 访问 HTTPS，导致证书和 SNI 问题。
- 认为同域名解析结果不同一定是错误，可能是就近调度。

## 自检清单

- 能否说明本地解析和递归解析流程。
- 能否解释 A、AAAA、CNAME、TTL。
- 能否用 `dig` 指定 DNS 服务器排查。
- 能否说明 JVM DNS 缓存风险。
- 能否区分 DNS 成功和 TCP/HTTPS 成功。
