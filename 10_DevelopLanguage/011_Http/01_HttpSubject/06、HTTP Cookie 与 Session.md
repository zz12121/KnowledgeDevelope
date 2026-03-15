# 六、HTTP Cookie 与 Session

###### 1. Cookie 的属性有哪些？

Cookie 是服务器通过 `Set-Cookie` 响应头下发给客户端、存储在浏览器中的小型数据片段。主要属性如下：

**基础属性**：
- `Name=Value`：Cookie 的名称和值，这是最核心的数据
- `Expires=日期`：绝对过期时间，到期后浏览器自动删除；不设置则是会话 Cookie（浏览器关闭就消失）
- `Max-Age=秒数`：相对过期时间，优先级高于 Expires（正数表示有效期，0 表示立即删除，负数表示会话）

**作用域**：
- `Domain=域名`：Cookie 对哪些域名有效（见第2题）
- `Path=路径`：Cookie 对哪些路径有效（见第2题）

**安全**：
- `Secure`：只通过 HTTPS 传输（见第3题）
- `HttpOnly`：禁止 JS 访问（见第3题）
- `SameSite`：跨站请求携带规则（见第4题）

示例：
```
Set-Cookie: sessionId=abc123; 
            Path=/; 
            Domain=example.com; 
            Max-Age=3600; 
            Secure; 
            HttpOnly; 
            SameSite=Lax
```

---

###### 2. Cookie 的 Domain 和 Path 属性是什么？

**Domain（域名范围）**：

指定哪些域名可以接收这个 Cookie。规则如下：
- 不设置：只有设置 Cookie 的那个具体域名可以访问
- 设置为 `Domain=example.com`：`example.com` 及其所有子域名（`api.example.com`、`static.example.com`）都能收到这个 Cookie
- **不能跨顶级域名**：example.com 不能设置 Domain 为 other.com

实际用途：在主域名和多个子域名之间共享登录状态（如 `app.example.com` 和 `admin.example.com` 共享 Cookie）。

**Path（路径范围）**：

指定哪些 URL 路径可以发送这个 Cookie：
- `Path=/`：整个网站所有路径（最常用）
- `Path=/api`：只有 `/api`、`/api/users` 等 `/api` 开头的路径才会携带

**匹配规则**：完整路径前缀匹配，`/api` 能匹配 `/api/users`，但不能匹配 `/apiv2`。

---

###### 3. Cookie 的 Secure 和 HttpOnly 属性是什么？

**Secure**：

标记后，这个 Cookie **只通过 HTTPS 连接发送**，HTTP 连接不会携带。

作用：防止 Cookie 在明文传输过程中被中间人窃取（网络嗅探攻击）。在生产环境，Session Cookie、Token Cookie 都应该设置 Secure。

**HttpOnly**：

标记后，**JavaScript 无法通过 `document.cookie` 读取或修改这个 Cookie**。

作用：防止 XSS 攻击——即使攻击者注入了恶意 JS 代码，也无法通过 JS 盗取 Cookie。

```javascript
// 设置了 HttpOnly 后，这行代码读不到该 Cookie
console.log(document.cookie);  // 不包含 HttpOnly 的 Cookie
```

**两者配合使用**是保护 Session Cookie 的标配：
```
Set-Cookie: JSESSIONID=xyz; Secure; HttpOnly; SameSite=Strict
```

**注意**：Secure 只防传输过程中被窃取，HttpOnly 只防 XSS 脚本读取，两者都无法防止 CSRF 攻击（需要 SameSite 或 CSRF Token）。

---

###### 4. 什么是 SameSite 属性？有哪些取值？

SameSite 控制跨站请求时是否携带 Cookie，是防御 CSRF 攻击的重要手段。

**三种取值**：

**`Strict`（最严格）**：只有同站请求才携带 Cookie，任何跨站请求都不携带。

问题：从外部链接进入网站时，浏览器不会携带 Cookie，用户必须重新登录。体验较差，适合银行、支付等高安全场景。

**`Lax`（宽松，Chrome 80+ 的默认值）**：大多数跨站请求不携带 Cookie，但以下情况例外（安全的导航方式）：
- `<a href>` 链接跳转
- `<form method="GET">` 表单提交
- 顶级窗口导航

POST 表单、`<iframe>`、`<img>`、AJAX 跨站请求都不携带。

**`None`（不限制）**：所有请求都携带 Cookie，跨站也一样。必须同时设置 `Secure`（Chrome 要求），否则会被忽略。

```
Set-Cookie: token=abc; SameSite=None; Secure
```

适用于确实需要跨站携带 Cookie 的场景，比如第三方嵌入组件、SSO 跨域认证。

---

###### 5. 如何防止 Cookie 被篡改？

Cookie 存在客户端，客户端是不可信的，攻击者可以手动修改 Cookie 内容。防篡改的核心思路是：**让服务器能检测到 Cookie 有没有被改过**。

**方案一：Cookie 签名（最常用）**

服务器在存 Cookie 时，用密钥对 Cookie 值做 HMAC 签名，把签名一起存入 Cookie：

```
// 存储：value.HMAC(secretKey, value)
Set-Cookie: userId=123.a3f2c8b91d...

// 验证：重新计算 HMAC，和 Cookie 里的签名对比
// 攻击者改了 userId=999，但无法伪造正确的 HMAC（不知道 secretKey）
```

Spring 中 `CookieSignedUtils` 或 Spring Security 都有这个机制。

**方案二：不在 Cookie 里存敏感数据**

只存一个无意义的 SessionID，真正的用户信息存在服务端（内存/Redis）。攻击者就算改了 SessionID，也无法映射到有效的用户数据。

**方案三：加密 Cookie**

用服务端密钥加密 Cookie 值，客户端看不到明文，也无从篡改。

**方案四：客户端验证 + HttpOnly**

设置 HttpOnly 防止 JS 读取，Secure 防止传输中被截取——虽然不能防止手动改，但减少了攻击面。

---

###### 6. Cookie 和 Token 有什么区别？

两者本质上都是"携带身份凭证"的机制，但实现方式和适用场景不同。

**Cookie**：
- 浏览器**自动管理**——每次请求自动携带，不需要手动处理
- 有完整的安全属性（HttpOnly/Secure/SameSite）
- 受**同源策略**和 **SameSite 属性**限制，天然有一定 CSRF 防护
- **大小限制**：通常 4KB
- **跨域限制**：需要 `CORS` 配合 `withCredentials` 才能跨域携带

**Token（通常是 JWT）**：
- 存储在 `localStorage` / `sessionStorage` / 内存中，**手动管理**（请求时手动加到 Header 里）
- 天然防 CSRF（不会被浏览器自动携带，XSS 才是威胁）
- 可携带用户信息，服务端无需查 DB 即可解析（无状态）
- **大小**：JWT 通常比 SessionID 大很多（几百字节到几KB）
- **跨域友好**：放在 `Authorization: Bearer <token>` 里，跨域没有额外限制

**选择建议**：
- 传统 Web（MVC 页面）：Cookie + Session，简单可靠
- 前后端分离、移动 App、微服务：JWT Token，跨域友好
- 高安全要求：Cookie + HttpOnly + SameSite=Strict（防 XSS 盗 Token）

---

###### 7. Session 是如何实现的？

Session 是"服务端保存用户状态"的机制，弥补了 HTTP 无状态的不足。

**基本流程**：

1. 用户登录，服务器验证成功后创建 Session 对象，存储用户信息（userId、权限等）
2. 服务器生成唯一的 SessionID（随机字符串），把 Session 数据存储在内存或 Redis 中
3. 服务器把 SessionID 通过 `Set-Cookie: JSESSIONID=xxx` 下发给客户端
4. 之后每次请求，浏览器自动携带 `Cookie: JSESSIONID=xxx`
5. 服务器根据 SessionID 查找对应的 Session 数据，恢复用户状态

**Session 存储方案**：
- **内存**（默认）：速度快，但不能水平扩展（多台服务器间 Session 不共享）
- **数据库**：持久化，但每次请求都要查 DB，性能差
- **Redis**（生产主流）：速度快、支持分布式、可以设过期时间——Spring Session + Redis 是标准方案

```java
// Spring Session + Redis 配置只需一个注解
@EnableRedisHttpSession(maxInactiveIntervalInSeconds = 1800)
public class SessionConfig {}
```

**Session 的安全注意**：
- SessionID 要足够随机（防猜测）
- 设置合理的过期时间（防会话劫持）
- Cookie 设置 HttpOnly + Secure

---

###### 8. Session 和 JWT 有什么区别？

这是前后端分离时代最常见的认证方案对比。

**Session（有状态）**：
- 用户信息存在服务端（内存/Redis），客户端只存一个 ID
- 服务端需要维护 Session 存储，有状态
- 优点：可以主动让 Session 失效（强制下线功能容易实现）
- 缺点：分布式环境需要 Session 共享（Redis），有额外依赖

**JWT（无状态）**：
- 用户信息编码在 Token 里（Base64），服务端不存储
- Token 由服务端用私钥签名，客户端无法伪造
- 优点：无状态，天然支持水平扩展；跨域友好；减轻服务端存储压力
- 缺点：**Token 一旦签发就无法提前吊销**（只能等自然过期），强制下线比较麻烦；Token 大小较大（每次请求都要带）

**JWT 主动失效方案**（解决无法吊销问题）：
- 黑名单（退出登录时把 Token 存 Redis，验证时检查黑名单）
- 短期 Token + Refresh Token（Access Token 有效期短，如 15 分钟）
- Token 版本号（存 DB，用户改密码时版本号+1，旧 Token 版本对不上就拒绝）

**选择**：微服务、跨域场景首选 JWT；传统 Web、需要强制下线功能首选 Session。

---

###### 9. 【追问】JWT Token 的结构是什么？如何保证安全？

**JWT 结构**（三段 Base64URL 编码用 `.` 拼接）：

```
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9    <- Header
.eyJ1c2VySWQiOjEyMywiZXhwIjoxNzQyMDAwMDAwfQ  <- Payload
.SflKxwRJSMeKKF2QT4fwpMeJf36POk6yJV_adQssw5c  <- Signature
```

**Header**：指定签名算法，如 `{"alg":"HS256","typ":"JWT"}`

**Payload**：存放声明（Claims），如：
```json
{
  "userId": 123,
  "roles": ["ADMIN"],
  "iat": 1742000000,   // 签发时间
  "exp": 1742003600    // 过期时间
}
```

**Signature**：用 Header 指定的算法 + 密钥对 `Header.Payload` 签名，防止篡改：
```
HMACSHA256(base64(header) + "." + base64(payload), secretKey)
```

**安全要点**：
- Payload 是 Base64 编码，**不是加密**，不要存密码、手机号等敏感信息
- 必须验证 `exp` 过期时间
- 密钥（secretKey）要足够随机且妥善保管（泄露就完了）
- 使用 HTTPS 传输，防止 Token 被截取
- Token 存在内存或 Cookie（HttpOnly），避免存 localStorage（XSS 可读取）
