# 01 HTTP 协议基础

> HTTP（HyperText Transfer Protocol）是应用层协议，运行在 TCP 之上（HTTP/3 改用 QUIC/UDP），是 Web 通信的基础。

## 1.1 核心特性速记

| 特性 | 说明 |
|------|------|
| 无状态 | 每次请求独立，不记忆上下文，靠 Cookie/Session/JWT 弥补 |
| 请求-响应模型 | 客户端主动，服务器被动 |
| 明文传输 | HTTP 不加密，HTTPS = HTTP + TLS |
| 灵活 | MIME 类型支持任意数据格式 |

## 1.2 报文结构

**请求报文**：
```
[请求行] GET /api/users HTTP/1.1
[Headers] Host: example.com
          Content-Type: application/json
[空行]
[Body]    {"name":"张三"}
```

**响应报文**：
```
[状态行] HTTP/1.1 200 OK
[Headers] Content-Type: application/json
[空行]
[Body]    {"code":0,"data":{}}
```

## 1.3 HTTP 方法语义

| 方法 | 幂等 | 安全 | 场景 |
|------|-----|------|------|
| GET | ✅ | ✅ | 查询 |
| HEAD | ✅ | ✅ | 只取 Header |
| OPTIONS | ✅ | ✅ | 查询支持的方法/CORS 预检 |
| PUT | ✅ | ❌ | 全量更新 |
| DELETE | ✅ | ❌ | 删除 |
| POST | ❌ | ❌ | 创建/提交 |
| PATCH | ❌（通常）| ❌ | 局部更新 |

## 1.4 常用 HTTP 状态码速查

**2xx 成功**：
- `200 OK`：请求成功，有响应体
- `201 Created`：创建成功（响应头 Location 指向新资源）
- `204 No Content`：成功但无响应体（DELETE/PUT 常用）
- `206 Partial Content`：分块响应（断点续传）

**3xx 重定向**：
- `301 Moved Permanently`：永久重定向（缓存，SEO 权重转移）
- `302 Found`：临时重定向（不缓存）
- `304 Not Modified`：协商缓存命中（不返回响应体）
- `307/308`：严格保持原请求方法的重定向

**4xx 客户端错误**：
- `400`：请求格式/参数错误
- `401`：未认证（没登录/Token 过期）
- `403`：无权限（认证了但没权限）
- `404`：资源不存在
- `405`：方法不允许
- `409`：并发冲突
- `429`：限流

**5xx 服务端错误**：
- `500`：通用服务器错误
- `502`：网关收到上游无效响应
- `503`：服务不可用（过载/维护）
- `504`：网关等待上游超时

## 1.5 常见请求头速查

| 请求头 | 作用 |
|--------|------|
| `Host` | 目标服务器域名（HTTP/1.1 必须） |
| `Authorization` | 认证凭证（`Bearer token` / `Basic base64`） |
| `Content-Type` | 请求体数据格式 |
| `Accept` | 期望的响应格式 |
| `Accept-Encoding` | 支持的压缩算法 |
| `Cookie` | 携带 Cookie |
| `User-Agent` | 客户端标识 |
| `Referer` | 请求来源页面 |
| `Origin` | 请求来源（跨域） |
| `Range` | 请求部分内容（断点续传） |
| `If-None-Match` | 协商缓存（ETag） |
| `If-Modified-Since` | 协商缓存（时间） |

## 1.6 常见响应头速查

| 响应头 | 作用 |
|--------|------|
| `Content-Type` | 响应体数据格式 |
| `Content-Length` | 响应体字节大小 |
| `Content-Encoding` | 压缩方式（gzip/br） |
| `Cache-Control` | 缓存策略 |
| `ETag` | 资源唯一标识（协商缓存） |
| `Last-Modified` | 最后修改时间（协商缓存） |
| `Set-Cookie` | 设置 Cookie |
| `Location` | 重定向目标 URL |
| `Access-Control-Allow-Origin` | CORS 允许的来源 |
| `Strict-Transport-Security` | HSTS（强制 HTTPS） |
| `X-Frame-Options` | 防点击劫持 |

## 1.7 MIME 类型速查

| 类型 | MIME |
|------|------|
| JSON | `application/json` |
| HTML | `text/html;charset=UTF-8` |
| CSS | `text/css` |
| JavaScript | `application/javascript` |
| 表单 | `application/x-www-form-urlencoded` |
| 文件上传 | `multipart/form-data` |
| 二进制下载 | `application/octet-stream` |
| PNG | `image/png` |
| JPEG | `image/jpeg` |
| WebP | `image/webp` |
| PDF | `application/pdf` |
| ZIP | `application/zip` |
