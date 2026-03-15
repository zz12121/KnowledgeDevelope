# 十、HTTP 认证与授权

###### 1. HTTP 有哪些认证方式？

HTTP 协议本身定义了几种认证方案，实际上还有很多应用层扩展方案：

**HTTP 标准认证方案**：
- **Basic 认证**：用户名+密码的 Base64 编码，简单但不安全（见第2题）
- **Digest 认证**：MD5 哈希挑战-响应机制，比 Basic 安全但有缺陷（见第3题）
- **Bearer 认证**：携带 Token（如 JWT）的现代方案（见第4题）
- **Negotiate**：Windows 环境的 NTLM/Kerberos 协商认证，企业内网常用

**应用层方案**：
- **Session Cookie**：登录后服务端维护 Session，客户端存 SessionID
- **OAuth 2.0**：授权框架，允许第三方应用代理访问（见第7题）
- **OpenID Connect**：基于 OAuth 2.0 的身份认证协议（加了 ID Token）
- **API Key**：静态密钥，通过 Header 或查询参数传递（机器间调用常用）

---

###### 2. 什么是 Basic 认证？

Basic 认证是最简单的 HTTP 认证方式：把"用户名:密码"用 Base64 编码后放在 `Authorization` 请求头里。

**流程**：
1. 客户端请求受保护资源
2. 服务器返回 `401 Unauthorized`，响应头带 `WWW-Authenticate: Basic realm="API"`
3. 客户端把"username:password"做 Base64 编码（如 `admin:123456` → `YWRtaW46MTIzNDU2`）
4. 客户端在每次请求时带上 `Authorization: Basic YWRtaW46MTIzNDU2`

```
Authorization: Basic YWRtaW46MTIzNDU2
```

**特点**：
- **不安全**：Base64 只是编码，不是加密，任何人都能解码得到明文密码——必须配合 HTTPS 使用
- **无状态**：每次请求都要带认证信息
- **实现简单**：Spring Security 几行配置就能开
- **用途**：内网管理接口、简单工具、测试环境；生产环境公共接口不推荐

---

###### 3. 什么是 Digest 认证？

Digest（摘要）认证是对 Basic 认证的改进，不直接传密码，而是用挑战-响应机制：

**流程**：
1. 服务器返回 401，带上随机挑战值（nonce）：
   ```
   WWW-Authenticate: Digest realm="api", nonce="abc123", algorithm=MD5
   ```
2. 客户端计算：`MD5(username:realm:password)` + nonce 等参数的混合哈希
3. 客户端发送哈希值，而不是密码本身

```
Authorization: Digest username="admin", realm="api", 
               nonce="abc123", uri="/login",
               response="6629fae49393a05397450978507c4ef1"
```

**对比 Basic 的改进**：即使请求被截获，攻击者只能看到哈希值，无法直接得到密码。

**但仍有局限**：
- 仍然容易受到中间人攻击（不能验证服务器身份）
- MD5 已被认为不安全（可以被暴力破解）
- 实现复杂，现代场景基本被 JWT Bearer 取代

---

###### 4. 什么是 Bearer 认证？

Bearer（持票人）认证是目前 RESTful API 最主流的认证方式，通过在 `Authorization` 头里携带 Token 来认证：

```
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
```

**"Bearer"的含义**：谁持有这个 Token，就有对应的权限——服务器不关心 Token 是谁的，只要 Token 有效就行（类似公交卡）。

**工作流程**：
1. 用户登录，服务器验证通过后颁发 Token（通常是 JWT）
2. 客户端把 Token 存起来（内存或 Cookie）
3. 每次请求在 Header 里带上 Token
4. 服务器验证 Token 的签名和有效期，决定是否放行

**和 Basic 认证的区别**：
- Bearer Token 有过期时间，Basic 没有
- Bearer Token 可以携带自定义信息（如用户角色），Basic 只有用户名密码
- Bearer Token 可以按需吊销（黑名单机制），Basic 只能改密码

**安全注意**：同样需要 HTTPS（防止 Token 在传输中被窃取）。

---

###### 5. Authorization 请求头的作用是什么？

`Authorization` 是客户端携带认证凭证的标准请求头，格式是：

```
Authorization: <认证方案> <凭证>
```

几种常见格式：
```
Authorization: Basic YWRtaW46cGFzc3dvcmQ=          # Basic 认证
Authorization: Bearer eyJhbGciOiJIUzI1NiJ9...      # Bearer Token（JWT）
Authorization: Digest username="admin", nonce="..." # Digest 认证
Authorization: AWS4-HMAC-SHA256 Credential=...      # AWS 签名认证
```

**API Key 变种**（非标准但常见）：
```
X-API-Key: your-api-key-here          # 自定义 Header
Authorization: ApiKey your-api-key     # Bearer 风格
```

**Spring Security 获取 Token**：
```java
@GetMapping("/protected")
public String protected(HttpServletRequest request) {
    String auth = request.getHeader("Authorization");
    if (auth != null && auth.startsWith("Bearer ")) {
        String token = auth.substring(7);
        // 验证 token...
    }
}
```

---

###### 6. WWW-Authenticate 响应头的作用是什么？

`WWW-Authenticate` 是服务器返回 `401 Unauthorized` 时必须携带的响应头，告诉客户端**应该用什么认证方案**来重新请求。

```
WWW-Authenticate: Basic realm="Admin Area"
WWW-Authenticate: Bearer realm="api", error="invalid_token", error_description="Token has expired"
WWW-Authenticate: Digest realm="api", nonce="abc123", algorithm=MD5
```

**字段说明**：
- 认证方案（Basic/Bearer/Digest）：告知客户端用哪种方式认证
- `realm`：认证区域的说明性文字（提示用户是哪个系统在要求登录）
- Bearer 方案还会带 `error` 和 `error_description` 说明认证失败的具体原因

**认证失败 vs 授权失败**：
- 没有凭证或凭证无效 → 服务器返回 401，必须带 `WWW-Authenticate` 头
- 有凭证但权限不足 → 服务器返回 403，不需要 `WWW-Authenticate`

---

###### 7. 什么是 OAuth 2.0？

OAuth 2.0 是**授权框架**，解决的核心问题是：让用户能够授权第三方应用访问自己在某服务上的资源，**而不需要把账号密码给第三方**。

**最典型的场景**：用微信/Google 账号登录第三方 App。

**四种角色**：
- **Resource Owner（资源所有者）**：用户本人
- **Client（客户端）**：第三方应用（想访问用户数据的那个）
- **Authorization Server（授权服务器）**：负责验证身份、颁发 Token（如微信的 OAuth 服务器）
- **Resource Server（资源服务器）**：存储用户数据的服务（如微信的用户信息接口）

**授权码模式（最安全，最常用）**：

```
用户 → 点击"微信登录" → 跳转到微信授权页面
微信 → 用户确认授权 → 回调第三方 App，带上 authorization_code
第三方 App → 用 code 换 access_token（这一步在后端进行，不暴露给前端）
第三方 App → 用 access_token 请求用户信息
```

**四种授权模式**：
- **Authorization Code**：授权码模式，最安全，适合有后端的 Web 应用
- **Implicit**：隐式模式，已废弃（不安全）
- **Client Credentials**：客户端凭证模式，机器间调用（没有用户参与）
- **Resource Owner Password**：密码模式，直接传用户名密码，不推荐

**OAuth 2.0 vs OpenID Connect**：OAuth 2.0 只是授权框架，不定义身份认证。OpenID Connect 在 OAuth 2.0 上加了 ID Token（JWT 格式），提供了"我是谁"的认证功能。

---

###### 8. 【追问】微服务架构中如何实现统一认证授权（网关鉴权模式）？

这是后端面试的高频实战题，结合 Spring Cloud Gateway + JWT 的方案是主流做法。

**架构设计**：

```
客户端
  ↓
API Gateway（统一鉴权）
  ├── 验证 JWT Token（签名、过期、黑名单）
  ├── 提取用户信息，写入请求 Header（X-User-Id、X-Roles）
  ├── 路由规则（权限路由：/admin/** 需要 ADMIN 角色）
  └── 转发到下游微服务（已带用户上下文）
  ↓
微服务 A / 微服务 B（业务层权限校验）
  ├── 读取 Header 中的用户信息（信任网关）
  └── 细粒度业务权限校验（如只能操作自己的数据）
```

**Spring Cloud Gateway 鉴权示例**：

```java
@Component
public class AuthFilter implements GlobalFilter {
    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        String token = exchange.getRequest().getHeaders()
                              .getFirst("Authorization");
        if (token == null || !token.startsWith("Bearer ")) {
            exchange.getResponse().setStatusCode(HttpStatus.UNAUTHORIZED);
            return exchange.getResponse().setComplete();
        }
        // 验证 JWT，提取用户信息
        Claims claims = jwtUtil.parseToken(token.substring(7));
        // 把用户信息透传给下游
        ServerHttpRequest mutatedRequest = exchange.getRequest().mutate()
            .header("X-User-Id", claims.getSubject())
            .header("X-Roles", claims.get("roles").toString())
            .build();
        return chain.filter(exchange.mutate().request(mutatedRequest).build());
    }
}
```

关联知识：[[27_SpringCloudKnowledge]]（Spring Cloud Gateway 章节）
