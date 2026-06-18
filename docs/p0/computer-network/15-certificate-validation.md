# 证书校验

## 面试定位

证书校验常和 HTTPS 握手一起考，也常出现在真实线上故障里。面试会问：浏览器或 Java 客户端如何校验证书、证书链是什么、根证书为什么可信、域名不匹配会怎样、证书过期如何排查。

核心回答：

客户端通过信任库中的根 CA 验证服务端证书链，再校验证书有效期、域名、用途、吊销状态等，最终确认服务端身份可信。

## 证书链

典型证书链：

```text
服务端证书
  -> 中间 CA 证书
  -> 根 CA 证书
```

服务端通常返回：

- 服务端证书。
- 一个或多个中间证书。

根证书通常已经内置在操作系统、浏览器或 JDK 信任库中，不需要服务端返回。

## 校验流程

客户端校验证书大致流程：

1. 构建证书链。
2. 用上级证书公钥验证下级证书签名。
3. 直到链路连接到本地信任库中的根 CA。
4. 校验证书是否在有效期内。
5. 校验证书域名是否匹配访问域名。
6. 校验证书用途是否允许服务端认证。
7. 可选校验证书是否被吊销。

任何一步失败，TLS 握手都可能失败。

## 域名校验

证书中用于域名匹配的字段主要是 SAN：

```text
Subject Alternative Name
DNS:example.com
DNS:*.example.com
```

现代客户端主要看 SAN，不应依赖 Common Name。

通配符注意：

- `*.example.com` 可匹配 `a.example.com`。
- 通常不匹配 `a.b.example.com`。
- 不匹配裸域 `example.com`，除非证书同时包含。

使用 IP 访问 HTTPS 服务时，如果证书没有包含该 IP，会校验失败。

## 有效期

证书包含：

- Not Before。
- Not After。

客户端本地时间如果错误，也可能导致“尚未生效”或“已过期”。

查看证书有效期：

```bash
echo | openssl s_client -connect example.com:443 -servername example.com 2>/dev/null | openssl x509 -noout -dates
```

## 信任库

不同客户端使用不同信任库：

- 浏览器可能使用自身或系统信任库。
- Java 默认使用 JDK 的 `cacerts`。
- 容器镜像可能缺少 CA 包。
- 企业内部证书需要导入内部根 CA。

Java 查看默认信任库：

```bash
keytool -list -cacerts
```

查看某个证书：

```bash
keytool -printcert -file server.crt
```

导入证书示例：

```bash
keytool -importcert -alias internal-ca -file ca.crt -keystore truststore.jks
```

生产环境要按组织安全规范管理 truststore，不要随意跳过校验。

## 吊销检查

证书吊销用于处理证书未过期但已不可信的情况。

常见机制：

- CRL：证书吊销列表。
- OCSP：在线证书状态协议。
- OCSP Stapling：服务端在握手中携带 OCSP 响应。

很多线上问题不是吊销导致，但面试中可以说明它是证书有效性检查的一部分。

## 常见错误

### 证书过期

表现：

```text
certificate has expired
```

处理：

- 更新服务端证书。
- 确认证书链完整。
- 检查客户端时间。

### 域名不匹配

表现：

```text
No subject alternative DNS name matching ...
```

处理：

- 使用证书覆盖的域名访问。
- 重新签发包含正确 SAN 的证书。
- 确认 SNI 是否正确。

### 证书链不完整

表现：

```text
unable to get local issuer certificate
```

处理：

- 服务端配置完整中间证书链。
- 确认客户端信任库包含根 CA。

### 自签名证书不可信

表现：

```text
self signed certificate
```

处理：

- 导入自签 CA 到信任库。
- 使用受信任 CA 签发证书。

## 命令示例

查看完整证书链：

```bash
openssl s_client -connect example.com:443 -servername example.com -showcerts
```

只看证书主题和颁发者：

```bash
echo | openssl s_client -connect example.com:443 -servername example.com 2>/dev/null | openssl x509 -noout -subject -issuer
```

查看 SAN：

```bash
echo | openssl s_client -connect example.com:443 -servername example.com 2>/dev/null | openssl x509 -noout -ext subjectAltName
```

用 curl 验证：

```bash
curl -v https://example.com/
curl -vk https://example.com/
```

`-k` 只用于排查，它会跳过证书校验。

## Java 排查

开启 SSL debug：

```bash
java -Djavax.net.debug=ssl,handshake,certpath ...
```

常见异常：

```text
PKIX path building failed
unable to find valid certification path to requested target
No subject alternative names matching IP address
```

排查点：

- JDK 版本是否过老。
- truststore 是否正确。
- 服务端是否返回中间证书。
- 访问域名是否匹配 SAN。
- 容器镜像是否缺 CA。

## 线上排查

### 浏览器正常，Java 失败

常见原因：

- 浏览器信任库和 JDK 信任库不同。
- JDK 版本老，缺少新根证书。
- Java 访问的是受限网络域名或 IP，证书不匹配。

### 部分机器失败

常见原因：

- 机器时间不一致。
- JDK 或系统 CA 包版本不同。
- 容器镜像不同。
- 发布时部分实例证书链配置不一致。

### 证书更新后失败

检查：

- 新证书是否包含所有域名。
- 中间证书是否配置完整。
- 客户端是否信任新 CA。
- 网关是否加载了新证书。

## 常见追问

### 根证书为什么可信

根证书预置在操作系统、浏览器或 JDK 信任库中，由客户端环境信任。信任根证书等于信任它签发或授权签发的证书链。

### 证书加密数据吗

证书本身不加密业务数据。证书用于身份认证和携带公钥，TLS 握手后协商出的对称密钥用于加密数据。

### 内部服务能不能跳过证书校验

不建议。内部网络也可能有中间人、误路由和代理风险。正确做法是配置内部 CA 和 truststore。

## 易错点

- 认为证书没过期就一定有效。
- 忽略 SAN 域名校验。
- 用 `curl -k` 证明 HTTPS 正常。
- 服务端只配置服务端证书，漏配中间证书。
- Java 服务里全局信任所有证书，埋下安全风险。

## 自检清单

- 能否说明证书链和根 CA 信任逻辑。
- 能否解释 SAN 域名匹配。
- 能否区分证书过期、链不完整、根不可信、域名不匹配。
- 能否用 `openssl s_client` 查看证书链和有效期。
- 能否排查 Java `PKIX path building failed`。
