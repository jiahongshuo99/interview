# HTTPS / TLS 握手

## 面试定位

HTTPS / TLS 握手是安全和网络结合的高频题。面试常问：HTTPS 比 HTTP 多了什么、TLS 握手做了哪些事、对称加密和非对称加密怎么配合、TLS 1.2 和 TLS 1.3 有什么区别、为什么证书能防中间人。

核心回答：

HTTPS 通常是 HTTP over TLS over TCP。TLS 握手负责协商协议版本、密码套件、校验证书、生成会话密钥，之后用对称加密保护 HTTP 数据。

## HTTPS 分层

```text
HTTP
TLS
TCP
IP
链路层
```

TLS 提供：

- 机密性：加密传输内容。
- 完整性：防止数据被篡改。
- 身份认证：通过证书确认服务端身份。

## 为什么既用非对称又用对称加密

非对称加密：

- 适合身份认证和密钥交换。
- 计算开销大。

对称加密：

- 加解密快。
- 适合大量数据传输。

TLS 的思路：

```text
握手阶段用非对称能力安全协商密钥
数据传输阶段用对称密钥加密 HTTP 数据
```

## TLS 1.2 握手简化流程

以 ECDHE 为例：

```text
ClientHello
  -> 支持的 TLS 版本
  -> 随机数
  -> 支持的密码套件
  -> SNI

ServerHello
  -> 选择的 TLS 版本
  -> 随机数
  -> 选择的密码套件
  -> 证书
  -> ECDHE 参数和签名

客户端校验证书和签名
双方基于 ECDHE 生成共享秘密
派生会话密钥

ChangeCipherSpec
Finished

开始加密传输 HTTP
```

关键点：

- 证书用于认证服务端身份。
- ECDHE 用于协商临时密钥，支持前向安全。
- Finished 消息用于验证握手过程未被篡改。

## TLS 1.3 简化

TLS 1.3 相比 TLS 1.2：

- 握手更少轮次，通常 1-RTT。
- 移除不安全或老旧算法。
- 默认使用支持前向安全的密钥交换。
- 加密更多握手内容。
- 支持 0-RTT，但 0-RTT 有重放风险。

简化流程：

```text
ClientHello，携带 key share
ServerHello，携带 key share 和证书相关消息
双方生成密钥
Finished
加密 HTTP 数据
```

## SNI

SNI 是 Server Name Indication，客户端在 TLS 握手中告诉服务端自己要访问哪个域名。

作用：

- 同一个 IP 承载多个 HTTPS 域名。
- 服务端根据 SNI 返回正确证书。

如果 SNI 不正确，可能出现：

- 证书域名不匹配。
- 网关路由错误。
- TLS 握手失败。

命令：

```bash
openssl s_client -connect 1.2.3.4:443 -servername example.com
```

## ALPN

ALPN 用于在 TLS 握手中协商应用层协议，例如：

- `http/1.1`
- `h2`

如果 HTTP/2 协商失败，可能降级到 HTTP/1.1。

查看：

```bash
openssl s_client -connect example.com:443 -servername example.com -alpn h2
```

## 会话复用

TLS 握手有开销，因此支持会话复用：

- Session ID。
- Session Ticket。
- TLS 1.3 PSK。

收益：

- 减少握手耗时。
- 降低 CPU。

注意：

- 会话复用不能替代 HTTP 连接池。
- 连接复用比重复握手更省。

## 命令示例

查看证书和握手：

```bash
openssl s_client -connect example.com:443 -servername example.com
```

查看 TLS 版本：

```bash
curl -v https://example.com/
```

指定 TLS 版本测试：

```bash
curl -v --tlsv1.2 https://example.com/
curl -v --tlsv1.3 https://example.com/
```

查看 HTTP/2 协商：

```bash
curl -v --http2 https://example.com/
```

抓握手：

```bash
sudo tcpdump -i any host example.com and tcp port 443 -w /tmp/tls.pcap
```

## 线上排查

### TLS 握手失败

常见原因：

- 证书过期。
- 证书链不完整。
- 域名不匹配。
- 客户端信任库缺根证书。
- TLS 版本不兼容。
- 密码套件不兼容。
- SNI 缺失或错误。
- 客户端时间错误。

排查：

```bash
openssl s_client -connect host:443 -servername domain
curl -vk https://domain/
```

### HTTPS 请求慢

拆分：

- DNS。
- TCP connect。
- TLS handshake。
- 服务端处理。
- 响应下载。

`curl` 查看耗时：

```bash
curl -o /dev/null -s -w 'dns=%{time_namelookup} connect=%{time_connect} tls=%{time_appconnect} total=%{time_total}\n' https://example.com/
```

### 证书发布后部分客户端失败

可能原因：

- 老 JDK 不支持新根证书。
- 中间证书链缺失。
- 使用 IP 访问导致域名不匹配。
- 老客户端不支持服务端只开启的 TLS 版本。

## 常见追问

### HTTPS 能防什么

能防窃听、篡改和伪造服务端身份。它不能防服务端自身被攻击，也不能自动解决业务鉴权问题。

### 证书里公钥是用来干什么

证书把域名和公钥绑定起来，并由 CA 签名。客户端验证证书后，可以信任该公钥属于目标域名。

### 前向安全是什么

即使服务端长期私钥未来泄露，历史会话密钥也不应被解密。ECDHE 这类临时密钥交换可以提供前向安全。

### TLS 握手完成后还用非对称加密吗

大量应用数据使用对称加密。非对称能力主要用于握手中的身份认证和密钥交换。

## 易错点

- 认为 HTTPS 是 HTTP 加了非对称加密传输所有数据。
- 忽略 SNI 导致的证书不匹配。
- 只看证书没过期，不看证书链是否完整。
- 把 TLS 握手耗时算到服务端业务耗时里。
- 认为 `curl -k` 成功就代表证书没问题。`-k` 跳过校验，只能用于临时排查。

## 自检清单

- 能否画出 HTTP、TLS、TCP 的层次。
- 能否说明 TLS 握手完成哪些目标。
- 能否解释对称加密和非对称加密的分工。
- 能否说明 SNI、ALPN、证书链的作用。
- 能否用 `openssl s_client` 排查握手和证书问题。
