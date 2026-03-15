# 04 HTTPS 与安全

## 4.1 HTTPS = HTTP + TLS

TLS 工作在传输层和应用层之间，提供三大保障：
- **机密性**：加密传输，第三方无法解读
- **完整性**：MAC 防止数据篡改
- **认证性**：证书验证服务器身份

**混合加密机制**：
- 非对称加密（RSA/ECDHE）：安全交换密钥（TLS 握手阶段）
- 对称加密（AES/ChaCha20）：加密实际传输数据（数据传输阶段）

---

## 4.2 TLS 握手过程（TLS 1.2）

```
客户端                          服务器
  │── Client Hello ────────────→│  发送支持的版本/加密套件/随机数
  │← Server Hello ──────────────│  选定版本和算法
  │← Certificate ───────────────│  服务器证书（含公钥）
  │← ServerHelloDone ───────────│
  │── 验证证书 ─────────────────×│  （本地操作）
  │── ClientKeyExchange ────────→│  发送 Pre-Master Secret（公钥加密）
  │── ChangeCipherSpec ─────────→│  切换加密通信
  │── Finished ────────────────→│
  │← ChangeCipherSpec ───────────│
  │← Finished ──────────────────│
  │═══════ 加密通信开始 ══════════│
```

**TLS 1.3 改进**：1-RTT 握手（更快），支持 0-RTT 会话恢复，移除不安全算法。

---

## 4.3 数字证书与 CA

**证书包含**：持有者域名、公钥、有效期、颁发 CA、CA 签名

**证书信任链**：
```
根 CA 证书（预装在 OS/浏览器）
  └── 中间 CA（根 CA 签名）
        └── 服务器证书（中间 CA 签名）
```

**证书类型**：
- DV（域名验证）：自动验证，几分钟，Let's Encrypt 免费颁发
- OV（组织验证）：需要几天，验证企业信息
- EV（扩展验证）：最严格，地址栏显示企业名，银行常用

**TLS 故障排查**：
```bash
openssl s_client -connect example.com:443      # 查看握手过程和证书
openssl s_client -connect example.com:443 -showcerts  # 显示证书链

# 验证证书和私钥是否匹配（MD5 要相同）
openssl x509 -noout -modulus -in cert.pem | md5sum
openssl rsa  -noout -modulus -in key.pem  | md5sum
```

---

## 4.4 HSTS（强制 HTTPS）

```
Strict-Transport-Security: max-age=31536000; includeSubDomains; preload
```

浏览器收到 HSTS 后，在 `max-age` 内所有访问自动变为 HTTPS，防止 SSL Stripping 攻击（中间人在首次 HTTP 请求时插入）。

**preload**：加入浏览器内置的 HSTS 预加载列表，首次访问也强制 HTTPS。

---

## 4.5 CORS 跨域

### 同源策略

同源 = 协议 + 域名 + 端口完全相同。不同源的 AJAX 请求被浏览器拦截（服务端已响应，只是浏览器不让 JS 读取）。

### 预检请求（OPTIONS）

非简单请求（有自定义 Header、使用 PUT/DELETE/JSON 等）在发真正请求前，浏览器先发 OPTIONS 预检：

```
OPTIONS /api/users HTTP/1.1
Origin: https://frontend.example.com
Access-Control-Request-Method: POST
Access-Control-Request-Headers: Authorization

← 204 No Content
Access-Control-Allow-Origin: https://frontend.example.com
Access-Control-Allow-Methods: GET, POST, PUT, DELETE
Access-Control-Max-Age: 86400    # 预检缓存 1 天，减少重复预检
```

### Spring Boot CORS 配置

```java
@Configuration
public class CorsConfig implements WebMvcConfigurer {
    @Override
    public void addCorsMappings(CorsRegistry registry) {
        registry.addMapping("/api/**")
            .allowedOrigins("https://app.example.com")
            .allowedMethods("GET", "POST", "PUT", "DELETE")
            .allowedHeaders("Authorization", "Content-Type")
            .allowCredentials(true)
            .maxAge(3600);
    }
}
```

**注意**：allowCredentials(true) 时，allowedOrigins 不能用 `*`，必须指定具体域名。

---

## 4.6 常见 Web 攻击防御

### CSRF（跨站请求伪造）

**原理**：攻击者诱导已登录用户访问恶意页面，以用户身份发送请求（利用浏览器自动携带 Cookie）。

**防御**：
1. **CSRF Token**（主流）：在表单/请求中带随机 Token，服务器验证（攻击者无法读取）
2. **SameSite=Lax/Strict**（最简单）：跨站请求不携带 Cookie
3. **验证 Origin/Referer**：检查请求来源
4. **双 Cookie 验证**：Cookie 和请求参数中带相同随机值

### XSS（跨站脚本攻击）

**类型**：反射型（URL 参数回显）/ 存储型（注入数据库）/ DOM 型（前端 JS 直接操作）

**防御**：
1. **输出编码**（最关键）：`<` → `&lt;`，现代框架（Vue/React）默认转义
2. **CSP 响应头**：`Content-Security-Policy: default-src 'self'`
3. **HttpOnly Cookie**：防止 XSS 盗 Cookie
4. **避免危险 API**：不用 `innerHTML = 用户输入`

### 点击劫持

**防御**：
```
X-Frame-Options: DENY                           # 禁止任何 iframe 嵌入
Content-Security-Policy: frame-ancestors 'none' # 同等效果（优先级更高）
```

### SQL 注入

**防御**：使用预编译语句（Prepared Statement）/ ORM 框架的参数绑定，**绝不拼接 SQL 字符串**。

```java
// 安全写法（MyBatis 的 #{} 参数）
SELECT * FROM users WHERE id = #{id}

// 危险写法（MyBatis 的 ${} 直接替换）
SELECT * FROM users WHERE id = ${id}  ← 有注入风险
```

关联知识：[[35_NettyKnowledge/05_协议与实战应用]]（TLS/SslHandler 实现）
