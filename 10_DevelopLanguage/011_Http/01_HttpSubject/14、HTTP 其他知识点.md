# 十四、HTTP 其他知识点

###### 1. 什么是 User-Agent？

`User-Agent`（简称 UA）是请求头字段，用于标识**发出请求的客户端软件信息**，包括浏览器类型、版本、操作系统等。

**典型的 User-Agent 值**：

```
// Chrome 浏览器（Windows）
Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/122.0.0.0 Safari/537.36

// Android 微信内置浏览器
Mozilla/5.0 (Linux; Android 14; Pixel 8) AppleWebKit/537.36 (KHTML, like Gecko) Version/4.0 Chrome/122.0.0.0 Mobile Safari/537.36 MicroMessenger/8.0.49

// Java HttpClient
Java/17.0.10

// curl
curl/8.1.2
```

**服务端用途**：
- **统计分析**：了解用户使用的浏览器/设备分布
- **内容适配**：桌面版 vs 移动版页面，或根据旧版浏览器给出降级提示
- **爬虫识别**：识别并拦截爬虫（Googlebot 有特定 UA）
- **区分接口调用方**：判断是浏览器请求还是 App 请求

**注意**：UA 可以被任意伪造，不能作为安全凭证——反爬虫时还需要结合行为分析、频率限制等手段。

---

###### 2. 什么是 Referer 请求头？

`Referer`（注意原文就是拼错的，历史遗留）是请求头，告诉服务器**当前请求是从哪个页面链接过来的**。

```
Referer: https://www.google.com/search?q=spring+boot
// 表示：用户是从 Google 搜索结果页点过来的
```

**服务端用途**：
- **防盗链**：图片、视频服务器检查 Referer，拒绝非本站请求（Nginx 的 `valid_referers`）
- **流量来源分析**：统计用户从哪里来（SEO 优化、广告投放效果分析）
- **CSRF 防御的辅助手段**：验证请求是否来自可信来源

**Referer 的限制**：
- HTTPS 页面跳转到 HTTP 页面时，浏览器不发送 Referer（安全降级）
- 可以通过 `Referrer-Policy` 响应头控制 Referer 发送策略
- 用户可以禁用 Referer 发送，因此不能作为唯一的 CSRF 防线

**Referrer-Policy 常见值**：

```
Referrer-Policy: no-referrer                  # 永不发送
Referrer-Policy: origin                        # 只发送源（域名），不发路径
Referrer-Policy: strict-origin-when-cross-origin  # 同源发完整，跨域只发域名（默认值）
Referrer-Policy: unsafe-url                   # 始终发完整 URL（不推荐）
```

---

###### 3. 什么是 Origin 请求头？

`Origin` 表示请求的**来源（协议 + 域名 + 端口）**，比 Referer 更简洁（只有源，没有路径信息）。

```
Origin: https://app.example.com
```

**和 Referer 的区别**：
- `Referer`：包含完整 URL（含路径和查询参数），历史遗留字段
- `Origin`：只包含源（protocol://domain:port），更安全（不暴露路径），CORS 专用

**何时发送 Origin**：
- 跨域请求（CORS）必须带
- POST 表单提交会带
- GET/HEAD 通常不带（除非是跨域）

**CORS 中的核心角色**：服务器根据 `Origin` 决定是否允许跨域，并在响应头里设置 `Access-Control-Allow-Origin`。

**安全用途**：和 Referer 类似，可以作为 CSRF 防御的参考——但同样可以被伪造（curl/Postman 可以随意设置），不能作为唯一安全手段。

---

###### 4. 什么是重定向？HTTP 如何实现重定向？

重定向是服务器告诉客户端：**你要的资源在另一个地址，去那里找吧**。

**实现方式**：服务器返回 3xx 状态码 + `Location` 响应头：

```
HTTP/1.1 301 Moved Permanently
Location: https://www.example.com/new-path
```

浏览器收到后，自动向 Location 指定的 URL 发新请求。

**常见场景**：

```
# HTTP → HTTPS 强制跳转（最常见）
http://example.com → 301 → https://example.com

# www 规范化
http://example.com → 301 → http://www.example.com

# 路径变更
/old-api/users → 301 → /api/v2/users

# 登录后跳转
GET /dashboard (未登录) → 302 → /login?redirect=/dashboard

# 支付成功跳转（Post/Redirect/Get 模式防重复提交）
POST /checkout → 303 → /order/success
```

**Spring Boot 重定向**：

```java
// Controller 重定向
return "redirect:/new-url";

// 或者用 HttpServletResponse
response.sendRedirect("https://example.com");

// 301 永久重定向
response.setStatus(HttpServletResponse.SC_MOVED_PERMANENTLY);
response.setHeader("Location", "/new-path");
```

**重定向链的风险**：多次重定向（`A → B → C → D`）会有多次往返开销，影响性能；如果形成循环重定向（`A → B → A`），浏览器会报错。

---

###### 5. Location 响应头的作用是什么？

`Location` 响应头用于两种场景：

**重定向（配合 3xx 状态码）**：

指定重定向的目标 URL，浏览器自动跳转：

```
HTTP/1.1 301 Moved Permanently
Location: https://www.new-address.com
```

**资源创建（配合 201 Created）**：

告诉客户端新创建资源的 URL：

```
POST /api/users HTTP/1.1
{...}

HTTP/1.1 201 Created
Location: https://api.example.com/api/users/123
```

这是 RESTful API 设计的规范做法，让调用方知道新资源在哪里，不用再查询。

---

###### 6. 什么是内容分发网络（CDN）的工作原理？

（详细的 CDN 介绍见第十二章第4题，这里补充技术细节）

**CDN 的核心调度机制**：

**DNS 调度（GSLB，全局服务器负载均衡）**：

1. 用户访问 `static.example.com`，DNS 查询
2. 权威 DNS 返回 CDN 的 CNAME：`static.example.com → cdn.provider.com`
3. 用户继续查询 `cdn.provider.com`
4. CDN 的 DNS 系统（GSLB）根据用户 IP 地理位置、节点负载、健康状态，返回最优节点 IP
5. 用户连接到该节点

**CDN 缓存层级**：

```
用户 → 边缘节点（最近的小节点）
       ↓（缓存未命中）
    区域汇聚节点（覆盖一个大区域）
       ↓（缓存未命中）
    中心节点（连接源站）
       ↓（缓存未命中）
    源站（你的服务器）
```

**缓存命中率优化**：
- 合理设置 `Cache-Control: max-age`（越长命中率越高，但更新延迟也越大）
- 使用内容 Hash 文件名（可以永久缓存）
- `CDN 刷新`（发布新版本后主动清除 CDN 缓存）
- `CDN 预热`（提前把内容推到 CDN 节点，避免新内容第一波请求打源站）

---

###### 7. 什么是 HTTP 隧道？

HTTP 隧道是通过 HTTP 协议**传输非 HTTP 数据**的技术，把其他协议的数据"包裹"在 HTTP 里传输。

**典型场景：HTTPS 代理（CONNECT 方法）**

```
客户端 → 代理服务器: CONNECT api.example.com:443 HTTP/1.1
代理服务器 → 目标服务器: 建立 TCP 连接
代理服务器 → 客户端: 200 Connection Established

[之后客户端直接通过代理的 TCP 连接和目标服务器做 TLS 握手]
[代理只转发字节流，不解析内容]
```

这就是 Charles、Fiddler 等抓包工具解密 HTTPS 的工作方式（它们会伪装成服务器，中间人模式）。

**其他应用场景**：
- VPN 穿越防火墙：在只开放了 80/443 端口的环境中，通过 HTTP 隧道传输其他协议
- SSH over HTTP：通过 HTTP 代理访问 SSH
- gRPC over HTTP/2：gRPC 把 Protocol Buffers 数据通过 HTTP/2 传输

**WebSocket 也是一种隧道**：从 HTTP 升级（Upgrade），之后通过同一 TCP 连接传输 WebSocket 帧数据。

---

###### 8. 如何调试 HTTP 请求？

调试 HTTP 请求的工具和方法：

**浏览器开发者工具（最常用）**：

Chrome DevTools Network 面板：
- 查看所有请求的 URL、方法、状态码、时序（Timing）
- 查看请求/响应的完整 Headers
- 查看响应体（支持 JSON 格式化）
- 网络节流（模拟慢网络环境）
- `Preserve log`（防止页面跳转时清空日志）

**Postman / Apifox**：

- 手动构造任意 HTTP 请求（设置 Headers、Body、认证）
- 支持环境变量管理不同环境的配置
- 可以保存和分享请求集合
- 支持自动化测试（Pre-request Script / Test Script）

**curl（命令行）**：

```bash
# 发 GET 请求，显示响应头
curl -v https://api.example.com/users

# 发 POST JSON 请求
curl -X POST https://api.example.com/users \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer your-token" \
  -d '{"name":"张三"}'

# 只看响应头
curl -I https://api.example.com/users

# 跟踪重定向
curl -L https://example.com
```

**抓包工具（Charles / Fiddler / Wireshark）**：

- Charles/Fiddler：代理模式，可以解密 HTTPS（需要安装 CA 证书）、修改请求/响应（Mock 接口）
- Wireshark：更底层，直接分析网络包，可以看 TCP 握手过程

**Spring Boot 应用调试**：

```yaml
# application.yml 开启 HTTP 请求日志
logging:
  level:
    org.springframework.web: DEBUG
    org.apache.http: DEBUG
```

---

###### 9. 【追问】如何设计一个高可用的 API 网关？

API 网关是微服务的统一入口，需要从多个维度保障高可用：

**功能层面**：
- **路由转发**：根据路径/Header 路由到不同微服务
- **认证鉴权**：统一 JWT 验证，不让无效请求到达后端
- **限流熔断**：防止流量冲击（令牌桶/漏桶算法），服务故障自动熔断
- **协议转换**：外部 REST → 内部 gRPC，外部 HTTP → 内部各种协议
- **日志链路追踪**：统一注入 TraceId

**高可用层面**：
- **多实例部署**：网关至少 2 个实例，Nginx/LB 做负载均衡
- **快速失败**：设置合理的超时时间（如 5s），下游慢不拖垮网关
- **缓存**：公共接口响应缓存在网关层
- **降级**：下游服务不可用时，网关返回降级响应而不是透传 500

**技术选型**：
- Spring Cloud Gateway（Java 技术栈，易集成 Spring Security/Sentinel）
- Kong（Nginx 内核，Lua 插件，性能好，云原生友好）
- Nginx（最轻量，适合简单场景）
- APISIX（开源，支持插件热更新）

关联知识：[[27_SpringCloudKnowledge]]
