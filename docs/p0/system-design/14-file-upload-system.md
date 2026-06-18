# 文件上传系统

## 面试定位

文件上传系统考察大文件处理、对象存储、断点续传、分片上传、秒传、权限、安全扫描、元数据一致性和带宽隔离。面试官希望你能把“上传文件”从应用服务里拆出来，避免应用机器承载大流量文件。

核心回答：

```text
应用服务负责鉴权、元数据和签名，文件流量直传对象存储；大文件走分片上传，完成后异步扫描和处理，元数据状态机保证一致性。
```

## 需求澄清

先问：

- 文件类型：图片、视频、文档、压缩包。
- 单文件大小上限和日上传量。
- 是否需要断点续传、分片上传、秒传。
- 文件是否公开访问，是否需要鉴权。
- 是否需要病毒扫描、内容审核、转码、缩略图。
- 文件保留多久，是否允许删除。
- 是否需要多地域访问和 CDN。
- 元数据一致性要求是什么。

## 核心模型

核心实体：

- 上传任务：uploadId、用户、文件名、大小、hash、状态。
- 文件元数据：fileId、storageKey、bucket、大小、类型。
- 分片记录：partNo、etag、大小、状态。
- 处理任务：扫描、转码、缩略图。

状态：

```text
INIT -> UPLOADING -> MERGING -> SCANNING -> AVAILABLE
INIT / UPLOADING -> FAILED
AVAILABLE -> DELETED
```

## 架构方案

推荐直传对象存储：

```text
客户端
  -> 请求创建上传任务
  -> 应用服务鉴权并返回预签名 URL
  -> 客户端直传对象存储
  -> 客户端通知上传完成
  -> 应用服务校验对象元数据
  -> 异步安全扫描/转码
  -> 文件状态 AVAILABLE
```

应用服务不转发文件流，避免带宽和磁盘成为瓶颈。

## 数据模型

### 文件元数据

```sql
CREATE TABLE file_meta (
    id BIGINT PRIMARY KEY,
    file_id VARCHAR(64) NOT NULL,
    owner_id BIGINT NOT NULL,
    original_name VARCHAR(512) NOT NULL,
    content_type VARCHAR(128) NOT NULL,
    file_size BIGINT NOT NULL,
    file_hash VARCHAR(128) DEFAULT NULL,
    bucket VARCHAR(128) NOT NULL,
    storage_key VARCHAR(512) NOT NULL,
    status VARCHAR(32) NOT NULL,
    created_at DATETIME NOT NULL,
    updated_at DATETIME NOT NULL,
    UNIQUE KEY uk_file_id (file_id),
    KEY idx_owner_created (owner_id, created_at),
    KEY idx_file_hash (file_hash)
);
```

### 上传任务

```sql
CREATE TABLE upload_session (
    id BIGINT PRIMARY KEY,
    upload_id VARCHAR(64) NOT NULL,
    file_id VARCHAR(64) NOT NULL,
    owner_id BIGINT NOT NULL,
    file_size BIGINT NOT NULL,
    file_hash VARCHAR(128) DEFAULT NULL,
    part_count INT NOT NULL,
    status VARCHAR(32) NOT NULL,
    expire_at DATETIME NOT NULL,
    created_at DATETIME NOT NULL,
    updated_at DATETIME NOT NULL,
    UNIQUE KEY uk_upload_id (upload_id),
    KEY idx_owner_created (owner_id, created_at)
);
```

### 分片记录

```sql
CREATE TABLE upload_part (
    id BIGINT PRIMARY KEY,
    upload_id VARCHAR(64) NOT NULL,
    part_no INT NOT NULL,
    part_size BIGINT NOT NULL,
    etag VARCHAR(128) DEFAULT NULL,
    status VARCHAR(32) NOT NULL,
    created_at DATETIME NOT NULL,
    updated_at DATETIME NOT NULL,
    UNIQUE KEY uk_upload_part (upload_id, part_no)
);
```

## 上传链路

### 小文件上传

```text
客户端请求上传
  -> 服务端鉴权、校验类型和大小
  -> 创建 file_meta INIT
  -> 返回预签名 PUT URL
  -> 客户端直传对象存储
  -> 客户端通知完成
  -> 服务端校验对象大小/hash
  -> 状态 SCANNING
  -> 扫描通过 AVAILABLE
```

### 大文件分片上传

```text
创建 upload_session
  -> 返回每个 part 的预签名 URL
  -> 客户端并发上传分片
  -> 记录 part etag
  -> complete multipart upload
  -> 校验最终对象
  -> 异步处理
```

分片大小常见 5MB 到 100MB，按对象存储限制和网络环境选择。

### 秒传

客户端先计算文件 hash：

```text
查询 file_hash 是否已有 AVAILABLE 文件
  -> 如果已有且权限允许，创建引用记录
  -> 如果没有，走正常上传
```

注意：

- hash 计算成本在客户端。
- 同 hash 文件也要考虑租户权限。
- 秒传不能绕过安全扫描，只有已扫描通过的文件才能复用。

## 下载链路

```text
用户请求下载
  -> 应用服务鉴权
  -> 检查文件状态 AVAILABLE
  -> 生成短期下载签名 URL
  -> 客户端/CDN 拉取对象存储
```

私有文件不要直接暴露永久 URL。

## 缓存、MQ 与一致性

缓存：

- 文件元数据可以短 TTL 缓存。
- 下载签名 URL 不宜长期缓存。
- CDN 缓存公开文件。

MQ：

- 病毒扫描。
- 图片压缩和缩略图。
- 视频转码。
- OCR/内容审核。
- 清理过期未完成上传任务。

一致性：

- DB 元数据是业务事实源。
- 对象存储是文件内容事实源。
- 上传完成后要校验对象存在、大小、hash。
- DB 已 AVAILABLE 但对象不存在时要告警和修复。
- 对象已上传但 DB 未完成时，过期清理。

## 容量估算

示例：

```text
日上传 100 万文件
平均 5MB
日新增存储 5TB
保留 1 年原始数据约 1.8PB
```

带宽：

```text
峰值上传 QPS 1000
平均文件 5MB
如果走应用转发，入口带宽约 5GB/s
```

这就是为什么要直传对象存储。

## 风险与降级

- 应用转发文件：带宽和磁盘打满。
- 文件类型伪造：校验 MIME 和文件头。
- 恶意文件：病毒扫描和内容审核。
- 分片未完成：过期清理。
- 重复上传浪费存储：hash 秒传和引用计数。
- 对象存储回调丢失：客户端完成通知 + 后台扫描校验。
- CDN 缓存私有文件：鉴权和短期签名。

降级：

- 转码、缩略图、OCR 延迟处理。
- 上传高峰限制大文件并发。
- 对象存储异常时暂停新上传，已上传文件查询可读缓存。

## 常见追问

### 为什么不让文件经过应用服务？

文件流量大，会占用应用带宽、线程、磁盘和 JVM 内存。应用服务应该只处理鉴权、签名、元数据，文件直传对象存储。

### 如何实现断点续传？

使用 uploadId 和分片记录。客户端查询已上传 part，缺哪个补哪个，最后 complete multipart upload。

### 如何保证 DB 和对象存储一致？

上传完成后校验对象元数据，再更新 DB 状态。定时任务扫描 INIT/UPLOADING 超时任务和 AVAILABLE 但对象缺失的异常。

### 文件删除怎么做？

先标记 DB 状态 DELETED，再异步删除对象。重要文件可延迟物理删除，防误删恢复。

### 如何防止非法文件？

上传前校验扩展名和大小，上传后校验文件头、MIME、病毒扫描、内容审核。未扫描通过前不允许公开下载。

## 自检清单

- 是否让文件直传对象存储。
- 是否有上传任务、文件元数据、分片记录。
- 是否支持大文件分片和断点续传。
- 是否支持秒传且不绕过安全扫描。
- 是否有上传完成校验。
- 是否通过 MQ 做扫描、转码、缩略图。
- 是否处理 DB 与对象存储一致性。
- 是否估算存储和带宽。
- 是否有鉴权、签名 URL、CDN 和删除策略。
