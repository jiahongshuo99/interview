# DNS 缓存

## 面试定位

DNS 缓存是线上切流、故障恢复和 Java 服务访问异常的常见根因。面试常问：DNS 缓存在哪些地方、TTL 是否一定生效、Java 为什么域名切换不生效、负缓存是什么、如何安全刷新 DNS。

核心回答：

DNS 缓存可能存在于浏览器、JVM、操作系统、容器 DNS、递归 DNS、网关和应用连接池中。TTL 只是权威记录给出的缓存建议，实际生效还受各层实现影响。

## 缓存位置

一次域名访问可能经过多层缓存：

```text
应用内缓存
  -> JVM DNS 缓存
  -> 操作系统 DNS 缓存
  -> hosts
  -> 本机 DNS stub
  -> 容器或节点 DNS
  -> 递归 DNS
  -> 权威 DNS
```

此外还有容易被忽略的“连接缓存”：

- HTTP 连接池已连接到旧 IP。
- RPC 客户端已维护旧连接。
- 网关 upstream 缓存旧解析。

即使 DNS 解析变了，已有长连接也不会自动迁移。

## TTL

TTL 表示记录建议缓存多久：

```bash
dig example.com
```

输出中第二列通常是剩余 TTL：

```text
example.com.  60  IN  A  1.2.3.4
```

注意：

- 递归 DNS 可能设置最小 TTL 或最大 TTL。
- JVM 可能忽略或覆盖 TTL。
- 负缓存也有 TTL。
- 连接池可能让旧 IP 继续被使用。

## 正缓存和负缓存

正缓存：域名解析成功后的结果缓存。

负缓存：解析失败结果也会缓存，例如 NXDOMAIN。

Java 中负缓存过长可能导致：

```text
域名刚创建
第一次解析失败
失败结果被缓存
后续一段时间仍解析失败
```

相关属性：

```text
networkaddress.cache.ttl
networkaddress.cache.negative.ttl
```

## Java DNS 缓存

JVM 会缓存 `InetAddress` 查询结果。常见参数：

```bash
-Dsun.net.inetaddr.ttl=60
-Dsun.net.inetaddr.negative.ttl=10
```

风险：

- TTL 太长：域名切流、故障摘除不及时。
- TTL 太短：DNS 查询压力增加，解析延迟增加。
- 负缓存太长：临时解析失败被放大。

生产建议：

- 对核心服务明确设置合理 TTL。
- 配合连接池最大生命周期，让旧 IP 连接逐步退出。
- 监控 DNS 解析失败和解析耗时。

## 操作系统缓存

不同系统差异很大。

Linux 可能由以下组件提供缓存：

- `systemd-resolved`
- `nscd`
- `dnsmasq`
- 容器环境中的 CoreDNS

macOS 刷新命令常见为：

```bash
sudo dscacheutil -flushcache
sudo killall -HUP mDNSResponder
```

Linux 需要看具体组件，例如：

```bash
resolvectl flush-caches
sudo systemctl restart nscd
sudo systemctl restart dnsmasq
```

生产环境不要随意重启 DNS 组件，先确认影响范围。

## 容器和 Kubernetes

Kubernetes 中常见链路：

```text
Pod
  -> /etc/resolv.conf
  -> CoreDNS Service
  -> upstream DNS
```

常见问题：

- CoreDNS 负载高。
- ndots 配置导致多次搜索域查询。
- 上游 DNS 慢。
- Pod 内 `/etc/resolv.conf` 与宿主机不同。
- Service 名解析和外部域名解析混在一起。

排查：

```bash
cat /etc/resolv.conf
nslookup service.namespace.svc.cluster.local
dig example.com
```

## DNS 缓存和连接池

DNS 只影响新建连接。已有 TCP 连接不会因为 DNS 记录变化自动断开。

例子：

```text
example.com 原来解析到 203.0.113.10
客户端连接池已建立到 203.0.113.10 的连接
DNS 切到 203.0.113.20
连接池仍可能复用 203.0.113.10 的旧连接
```

治理：

- 设置连接最大生命周期。
- 故障切流时主动关闭旧连接池。
- 客户端定期重新解析。
- 注册中心场景使用实例变更推送。

## 命令示例

查看当前解析和 TTL：

```bash
dig example.com
dig +nocmd example.com +noall +answer
```

指定 DNS 服务器：

```bash
dig @8.8.8.8 example.com
```

绕过 DNS 连接指定 IP：

```bash
curl -v --resolve example.com:443:1.2.3.4 https://example.com/
```

查看 Java 进程参数：

```bash
jcmd <pid> VM.system_properties | grep -i inetaddr
```

## 线上排查

### 域名切流后仍访问旧 IP

可能原因：

- JVM 正缓存未过期。
- 递归 DNS 缓存未过期。
- HTTP 连接池复用旧连接。
- 网关或代理缓存 upstream。
- 客户端使用本地 hosts。

排查：

- 在客户端机器上 `dig`。
- 指定不同 DNS 服务器对比。
- 看连接对端 IP。
- 重启或刷新连接池验证。

### 部分机器解析失败

可能原因：

- 机器 DNS 配置不同。
- 某个递归 DNS 异常。
- 负缓存存在。
- 容器 DNS 异常。

排查：

```bash
cat /etc/resolv.conf
dig example.com
dig @nameserver example.com
```

### DNS QPS 异常升高

可能原因：

- TTL 设置太短。
- 应用禁用缓存。
- 连接池频繁新建连接。
- 大量短生命周期容器启动。

治理：

- 合理 TTL。
- 本地 DNS 缓存。
- 客户端连接复用。
- 减少不必要解析。

## 常见追问

### TTL 到期后一定会重新查权威 DNS 吗

不一定。客户端通常问递归 DNS，递归 DNS 缓存到期后才会继续查权威 DNS。各层还可能有额外缓存策略。

### 修改 hosts 后 Java 服务一定立即生效吗

不一定。JVM 可能已经缓存旧解析，连接池也可能已有旧连接。通常需要清理缓存、重启进程或等待 TTL，并关闭旧连接。

### DNS 缓存如何影响故障恢复

故障 IP 从 DNS 摘除后，缓存未过期的客户端仍可能访问故障 IP，导致恢复滞后。

## 易错点

- 认为 TTL 是所有客户端严格遵守的硬规则。
- 忽略负缓存。
- 忽略连接池旧连接。
- 只在本机 dig 正常，就认为所有机器正常。
- 不区分操作系统缓存、JVM 缓存和递归 DNS 缓存。

## 自检清单

- 能否列出 DNS 缓存可能存在的层次。
- 能否解释正缓存和负缓存。
- 能否说明 JVM DNS 缓存和连接池的关系。
- 能否排查切流后仍访问旧 IP。
- 能否用 `dig @server`、`curl --resolve` 做验证。
