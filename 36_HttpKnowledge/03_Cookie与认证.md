# 03 Cookie 与认证

## 3.1 Cookie 核心属性速查

```
Set-Cookie: sessionId=abc123; 
            Max-Age=3600;           # 有效期（秒），优先级高于 Expires
            Domain=example.com;     # 有效域名（含子域名）
            Path=/;                 # 有效路径
            Secure;                 # 仅 HTTPS 传输
            HttpOnly;               # 禁止 JS 读取（防 XSS 盗 Cookie）
            SameSite=Lax            # 跨站请求携带规则
```

### SameSite 三种取值

| 值 | 跨站 POST 携带 | 跨站 GET 链接 | 场景 |
|----|--------------|--------------|------|
| `Strict` | ❌ | ❌ | 高安全场景（银行） |
| `Lax`（Chrome 默认） | ❌ | ✅ | 通用场景 |
| `None` | ✅（必须加 Secure） | ✅ | 第三方嵌入/SSO |

---

## 3.2 Session vs JWT 对比

| 维度 | Session | JWT |
|------|---------|-----|
| 存储 | 服务端（内存/Redis） | 客户端（Cookie/localStorage） |
| 状态 | 有状态 | 无状态 |
| 主动失效 | 容易（删除 Session） | 困难（需要黑名单或版本号） |
| 水平扩展 | 需要 Session 共享（Redis） | 天然支持 |
| 大小 | SessionID 小 | JWT 较大（几百字节-几KB） |
| 跨域 | 需要配合 CORS withCredentials | 放在 Header 里，天然跨域 |

**选择建议**：
- 传统 Web + 需要强制下线 → Session + Redis
- 前后端分离 + 微服务 → JWT
- 高安全需求 → Cookie + HttpOnly + SameSite=Strict（防 XSS 盗 Token）

### Spring Session + Redis 配置

```java
@EnableRedisHttpSession(maxInactiveIntervalInSeconds = 1800)
public class SessionConfig {}
```

### JWT 结构

```
Header.Payload.Signature

Header: {"alg":"HS256","typ":"JWT"}
Payload: {"userId":123,"roles":["ADMIN"],"exp":1742003600}
Signature: HMACSHA256(base64(header)+"."+base64(payload), secretKey)
```

**安全注意**：
- Payload 是 Base64，**不是加密**，不要存密码等敏感信息
- 必须验证 `exp` 过期时间
- 用 HTTPS 传输，防止 Token 被截取
- 避免存 localStorage（XSS 可读取），优先存 Cookie（HttpOnly）

### JWT 主动失效方案

```
方案1: Redis 黑名单（退出登录时写入，验证时检查）
方案2: Token 版本号（存DB，改密码时版本+1，旧Token拒绝）
方案3: 短期 Access Token（15分钟）+ 长期 Refresh Token（7天）
```

---

## 3.3 HTTP 认证方案

### Basic 认证

```
Authorization: Basic YWRtaW46cGFzc3dvcmQ=    # base64("admin:password")
WWW-Authenticate: Basic realm="Admin Area"    # 401 时服务器返回
```

**特点**：简单但不安全（Base64 可逆），必须配合 HTTPS，适合内网管理接口。

### Bearer Token 认证（主流）

```
Authorization: Bearer eyJhbGciOiJIUzI1NiJ9...
```

登录后服务器颁发 JWT，每次请求在 Header 里携带。

### OAuth 2.0 授权码模式（第三方登录）

```
用户点击"微信登录"
  → 跳转微信授权页（带 redirect_uri、client_id）
  → 用户授权，微信回调，带 authorization_code
  → 后端用 code 换 access_token（后端完成，不暴露给前端）
  → 用 access_token 获取用户信息
```

四种模式：授权码（最安全）/ 隐式（废弃）/ 客户端凭证（机器间）/ 密码（不推荐）

---

## 3.4 网关统一鉴权模式（微服务）

```java
// Spring Cloud Gateway 全局过滤器
@Component
public class AuthFilter implements GlobalFilter {
    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        String token = exchange.getRequest().getHeaders().getFirst("Authorization");
        // 验证 JWT，提取用户信息，透传给下游 Header
        ServerHttpRequest mutated = exchange.getRequest().mutate()
            .header("X-User-Id", userId)
            .header("X-Roles", roles)
            .build();
        return chain.filter(exchange.mutate().request(mutated).build());
    }
}
```

**架构**：API Gateway 统一鉴权 → 下游微服务信任 Header 中的用户信息 → 业务层细粒度权限校验

关联知识：[[27_SpringCloudKnowledge]]
