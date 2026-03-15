# 二、HTTP 请求与响应

###### 1. HTTP 请求报文的结构是什么？

HTTP 请求报文由三部分组成：**请求行 + 请求头 + 请求体**，头部和正文之间必须有一个空行。

```
GET /api/users?page=1 HTTP/1.1        ← 请求行：方法 + 请求URI + 协议版本
Host: api.example.com                 ← 请求头开始
Accept: application/json
Authorization: Bearer eyJhbGc...
User-Agent: Mozilla/5.0
                                      ← 空行（必须有）
                                      ← GET 请求通常没有请求体
```

POST 请求示例（有请求体）：

```
POST /api/users HTTP/1.1
Host: api.example.com
Content-Type: application/json
Content-Length: 45

{"name":"张三","email":"zhangsan@demo.com"}
```

**请求行**三要素：
- 请求方法（GET/POST/PUT/DELETE 等）
- 请求 URI（路径 + 查询参数）
- HTTP 版本（HTTP/1.1 或 HTTP/2）

---

###### 2. HTTP 响应报文的结构是什么？

响应报文同样是三部分：**状态行 + 响应头 + 响应体**。

```
HTTP/1.1 200 OK                       ← 状态行：协议版本 + 状态码 + 状态描述
Content-Type: application/json        ← 响应头
Content-Length: 58
Cache-Control: no-cache
Date: Sun, 15 Mar 2026 08:00:00 GMT
                                      ← 空行
{"code":0,"data":{"id":1,"name":"张三"}}  ← 响应体
```

**状态行**三要素：
- HTTP 版本
- 状态码（三位数字，如 200、404、500）
- 状态描述（OK、Not Found、Internal Server Error）

响应体的内容类型由 `Content-Type` 决定，可以是 JSON、HTML、图片、文件流等任意格式。

---

###### 3. 什么是 HTTP 请求头？常见的请求头有哪些？

请求头是附在请求报文里的键值对，用来传递客户端信息、控制请求行为、协商内容格式等。

常见请求头分类：

**内容相关**：
- `Content-Type`：请求体的数据格式（如 `application/json`、`multipart/form-data`）
- `Content-Length`：请求体的字节大小
- `Accept`：客户端能接受的响应内容类型（如 `application/json`）
- `Accept-Encoding`：支持的压缩算法（如 `gzip, br`）
- `Accept-Language`：期望的语言（如 `zh-CN,zh;q=0.9`）

**认证相关**：
- `Authorization`：认证凭证（如 `Bearer <token>` 或 `Basic <base64>`）
- `Cookie`：携带 Cookie 数据

**连接相关**：
- `Host`：目标服务器域名（HTTP/1.1 必须有）
- `Connection`：连接类型（`keep-alive` 或 `close`）
- `User-Agent`：客户端标识（浏览器类型、版本等）

**缓存相关**：
- `Cache-Control`：缓存控制指令
- `If-None-Match`：条件请求，携带 ETag 值
- `If-Modified-Since`：条件请求，携带上次修改时间

**其他**：
- `Referer`：请求来源页面 URL
- `Origin`：请求来源（跨域请求使用）
- `X-Forwarded-For`：经过代理后的真实客户端 IP

---

###### 4. 什么是 HTTP 响应头？常见的响应头有哪些？

响应头是服务器在响应报文里附带的键值对，用来描述响应体格式、控制缓存、设置 Cookie、指定安全策略等。

**内容相关**：
- `Content-Type`：响应体的数据格式（如 `application/json;charset=UTF-8`）
- `Content-Length`：响应体字节大小
- `Content-Encoding`：响应体压缩方式（如 `gzip`）
- `Transfer-Encoding`：传输编码方式（如 `chunked` 分块传输）

**缓存相关**：
- `Cache-Control`：缓存策略（如 `max-age=3600`, `no-cache`）
- `ETag`：资源唯一标识（用于协商缓存）
- `Last-Modified`：资源最后修改时间
- `Expires`：资源过期时间（绝对时间，HTTP/1.0 的产物）

**安全相关**：
- `Set-Cookie`：设置客户端 Cookie
- `Strict-Transport-Security`（HSTS）：强制使用 HTTPS
- `X-Content-Type-Options: nosniff`：禁止 MIME 类型嗅探
- `X-Frame-Options`：防止点击劫持
- `Content-Security-Policy`（CSP）：控制可加载的资源来源

**跨域相关**：
- `Access-Control-Allow-Origin`：允许的跨域来源
- `Access-Control-Allow-Methods`：允许的请求方法

**其他**：
- `Location`：重定向目标 URL（配合 3xx 状态码使用）
- `Server`：服务器软件信息（如 `nginx/1.21.0`）
- `Date`：响应生成时间

---

###### 5. HTTP 请求方法 PUT 和 PATCH 有什么区别？

两者都是更新资源，但语义不同：

**PUT**（全量更新）：用发送的数据**完整替换**目标资源。如果请求体里只有部分字段，服务器应该把其他字段清空。

```
PUT /api/users/1
{"name":"张三","email":"new@example.com","age":25}
// 用这整个对象替换 id=1 的用户数据，原来有其他字段会被覆盖
```

**PATCH**（局部更新）：只更新请求体里指定的字段，其他字段保持不变。

```
PATCH /api/users/1
{"email":"new@example.com"}
// 只改邮箱，其他字段不动
```

实际项目中 PATCH 更常用，因为大多数场景都是修改某个字段，而不是整体替换。不过很多团队图省事直接用 PUT，语义上不太严格，只要团队内统一就行。

**幂等性**：PUT 是幂等的（多次调用结果一样），PATCH 理论上可以是幂等的也可以不是（取决于实现，比如"将 age 增加 1"就不幂等）。

---

###### 6. HTTP 请求方法 DELETE 的作用是什么？

DELETE 用于**删除指定资源**，是 RESTful API 的四大基础方法（GET/POST/PUT/DELETE）之一。

```
DELETE /api/users/1         // 删除 id=1 的用户
DELETE /api/orders/100      // 删除订单 100
```

几个要注意的点：
- **幂等性**：DELETE 是幂等的——删一次和删多次结果一样（资源都不存在了）。但第一次返回 200，之后再删通常返回 404。
- **有无请求体**：HTTP 规范没有禁止 DELETE 带请求体，但大多数服务器和框架不推荐，也很少这么用。
- **安全性**：DELETE 是"不安全"的方法（会改变服务器状态），因此不应该被缓存，也不能随意重试。
- **软删除**：实际业务中很多时候 DELETE 接口实现的是"软删除"（标记 is_deleted=1），并不真正删除数据。

---

###### 7. HTTP 请求方法 HEAD 的作用是什么？

HEAD 和 GET 几乎一样，区别是：**响应只有头部，没有响应体**。

使用场景：
- **检查资源是否存在**：发 HEAD 请求，看返回 200 还是 404，不用传输整个响应体
- **获取资源元信息**：通过 `Content-Length` 知道文件大小、`Content-Type` 知道文件类型、`Last-Modified` 知道最后修改时间——下载前先"探路"
- **断点续传前检测**：用 HEAD 获取文件总大小，再按 Range 分段下载

```
HEAD /files/large-video.mp4 HTTP/1.1
Host: cdn.example.com

// 响应：
HTTP/1.1 200 OK
Content-Length: 524288000    // 500MB
Content-Type: video/mp4
Last-Modified: Sun, 15 Mar 2026 06:00:00 GMT
Accept-Ranges: bytes         // 支持断点续传
```

HEAD 请求必须和 GET 请求的响应头完全一致（除了没有 Body）——这是规范要求，方便客户端判断。

---

###### 8. HTTP 请求方法 OPTIONS 的作用是什么？

OPTIONS 用于**查询服务器支持哪些 HTTP 方法**，主要有两个实际用途：

**用途一：CORS 预检请求（最常见）**

跨域时，浏览器在发真正请求前，会先发一个 OPTIONS 预检请求，询问服务器是否允许跨域：

```
OPTIONS /api/users HTTP/1.1
Origin: https://frontend.example.com
Access-Control-Request-Method: POST
Access-Control-Request-Headers: Authorization, Content-Type
```

服务器响应：

```
HTTP/1.1 204 No Content
Access-Control-Allow-Origin: https://frontend.example.com
Access-Control-Allow-Methods: GET, POST, PUT, DELETE
Access-Control-Allow-Headers: Authorization, Content-Type
Access-Control-Max-Age: 86400    // 预检结果缓存 24 小时
```

**用途二：查询服务器能力**

向 `*` 发送 OPTIONS 请求（`OPTIONS * HTTP/1.1`）可以查询整个服务器支持的方法，响应头里 `Allow` 字段会列出来。

---

###### 9. HTTP 请求方法 TRACE 和 CONNECT 是什么？

这两个方法比较特殊，实际业务开发中几乎不会手写：

**TRACE（回显诊断）**：

服务器把收到的请求原样回显给客户端，主要用于调试，看请求经过各级代理后有没有被修改。返回的 `Content-Type` 是 `message/http`。

但 TRACE 有安全风险（XST 跨站追踪攻击），现代 Web 服务器默认都会禁用它，生产环境不要开启。

**CONNECT（建立隧道）**：

用于建立 TCP 隧道，最典型的场景是 **HTTPS 代理**。客户端先发 CONNECT 请求给代理服务器，让代理和目标服务器建立 TCP 连接，之后客户端和目标服务器之间的通信就直接通过这个隧道，代理只负责转发字节流。

```
CONNECT api.example.com:443 HTTP/1.1
Host: api.example.com:443

// 代理响应：
HTTP/1.1 200 Connection Established
// 之后直接走 TLS 握手，代理不解密
```

---

###### 10. 【追问】GET 和 POST 有哪些本质区别？

这是面试中出镜率最高的 HTTP 问题，答案要超越"GET 传参在 URL，POST 传参在 Body"这个层次：

**语义区别**：
- GET：读取/查询资源，不改变服务器状态（安全且幂等）
- POST：提交数据，通常会改变服务器状态（不安全，不幂等）

**技术区别**：

| 维度 | GET | POST |
|------|-----|------|
| 参数位置 | URL 查询字符串 | 请求体（Body） |
| 数据长度 | 受 URL 长度限制（约 2KB~8KB，浏览器和服务器各有限制） | 理论无限制（服务器可配置） |
| 安全性 | 参数明文暴露在 URL 中，会被浏览器历史、服务器日志记录 | 参数在 Body 中，不会出现在日志/历史记录（但不加密同样可被抓包） |
| 幂等性 | 幂等（多次请求结果相同） | 不幂等（多次提交可能创建多条数据） |
| 缓存 | 可被浏览器缓存 | 默认不缓存 |
| 书签/分享 | URL 带参数，可直接分享链接 | 不能分享 |

**关键认知**：POST 并不比 GET 安全——二者都是明文传输，都能被抓包。真正的安全要靠 HTTPS，而不是换成 POST。
