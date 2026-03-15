# SpringBoot 安全与认证

## 一、Spring Security 集成

Spring Boot 通过自动配置简化了 Spring Security 集成，引入 `spring-boot-starter-security` 后自动开启全路径认证保护、提供默认表单登录页。

**核心配置方式（现代推荐）：**

```java
@Configuration
@EnableWebSecurity
public class SecurityConfig {

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
            .authorizeHttpRequests(authz -> authz
                .requestMatchers("/public/**").permitAll()
                .anyRequest().authenticated()
            )
            .formLogin(withDefaults())
            .httpBasic(withDefaults());
        return http.build();
    }
}
```

**自动配置机制：** `@EnableWebSecurity` 导入 `WebSecurityConfiguration`，构建 `FilterChainProxy`（核心过滤器链）。`SecurityFilterChain` Bean 定义 URL 与安全规则的映射关系，最终被注入到 `FilterChainProxy` 中拦截所有请求。

**用户认证源配置：**

- **内存认证**：`InMemoryUserDetailsManager`，适合演示和测试
- **数据库认证**：实现 `UserDetailsService` 接口，重写 `loadUserByUsername()` 从数据库加载用户信息
- **密码编码器**：必须配置 `BCryptPasswordEncoder` Bean，生产环境禁止明文密码

---

## 二、RBAC 权限控制

RBAC 核心思想：权限赋予角色，用户通过角色间接获得权限。

**数据模型：** User ↔ Role ↔ Permission，多对多关系。

**URL 级别权限控制：**

```java
http.authorizeHttpRequests(authz -> authz
    .requestMatchers("/admin/**").hasRole("ADMIN")
    .requestMatchers("/api/**").hasAuthority("API_WRITE")
    .anyRequest().authenticated()
);
```

**方法级别权限控制（更精细）：**

```java
@Configuration
@EnableMethodSecurity(prePostEnabled = true)
public class MethodSecurityConfig {}

@Service
public class SomeService {
    @PreAuthorize("hasRole('ADMIN') or #userId == authentication.principal.id")
    public User getUserProfile(Long userId) { ... }
    
    @PostAuthorize("returnObject.owner == authentication.name")
    public SensitiveData getData() { ... }
}
```

**底层机制：** `@EnableMethodSecurity` 注册 AOP 顾问，方法调用前通过 `PreInvocationAuthorizationAdvice` 评估 SpEL 表达式决定是否放行。

---

## 三、JWT 认证

JWT 是无状态认证机制，所有必要信息都包含在令牌本身，服务器不存储会话信息。

**JWT 认证流程：**

1. 客户端发送用户名/密码登录，服务端验证通过后签发 JWT
2. 客户端后续请求在 `Authorization: Bearer <token>` 头中携带 JWT
3. 服务端通过自定义过滤器验证 JWT，提取用户信息注入 SecurityContext

**自定义 JWT 过滤器核心逻辑：**

```java
public class JwtAuthenticationFilter extends OncePerRequestFilter {
    @Override
    protected void doFilterInternal(HttpServletRequest request, ...) {
        String authHeader = request.getHeader("Authorization");
        if (authHeader != null && authHeader.startsWith("Bearer ")) {
            String jwt = authHeader.substring(7);
            String username = jwtUtil.extractUsername(jwt);
            
            if (username != null && SecurityContextHolder.getContext().getAuthentication() == null) {
                UserDetails userDetails = userDetailsService.loadUserByUsername(username);
                if (jwtUtil.validateToken(jwt, userDetails)) {
                    // 将认证信息写入 SecurityContext，表示本次请求已认证
                    UsernamePasswordAuthenticationToken authToken =
                        new UsernamePasswordAuthenticationToken(userDetails, null, userDetails.getAuthorities());
                    SecurityContextHolder.getContext().setAuthentication(authToken);
                }
            }
        }
        filterChain.doFilter(request, response);
    }
}
```

**Spring Security 配置关键点：**

- 禁用 CSRF（JWT 不依赖 Cookie，CSRF 威胁较小）
- 会话策略设为 `STATELESS`
- 在 `UsernamePasswordAuthenticationFilter` 前插入 JWT 过滤器

---

## 四、JWT vs Session

| 维度 | Session | JWT |
|------|---------|-----|
| 状态 | 有状态，服务端存储会话数据 | 无状态，信息编码在令牌中 |
| 存储位置 | 服务端（内存/Redis），客户端存 Session ID | 客户端（Header/LocalStorage/Cookie） |
| 水平扩展 | 需要 Redis 等集中式会话存储 | 天然支持分布式，任意实例可验证 |
| 注销 | 立即生效，删除服务端记录即可 | 困难，需要黑名单或短过期时间 |
| 安全风险 | CSRF 攻击（Cookie 自动发送） | XSS 攻击（Payload 仅 Base64，非加密） |

**选型建议：**

- **Session**：传统服务端渲染 Web 应用、需要立即撤销权限的场景
- **JWT**：前后端分离 API、微服务架构、移动端、SSO 场景

---

## 五、OAuth 2.0

OAuth 2.0 是授权框架，允许第三方应用在获得用户授权后有限度地访问用户资源，而无需暴露密码。

**四个核心角色：** 客户端（第三方应用）、资源所有者（用户）、授权服务器、资源服务器。

**Spring Boot 作为 OAuth2 客户端（集成第三方登录）：**

```yaml
spring:
  security:
    oauth2:
      client:
        registration:
          github:
            client-id: your-client-id
            client-secret: your-client-secret
            scope: user:read
```

```java
http.oauth2Login(withDefaults());  // 启用 OAuth2 登录
```

**Spring Boot 作为 OAuth2 资源服务器（保护自己的 API）：**

```java
http.oauth2ResourceServer(oauth2 -> oauth2.jwt(withDefaults()));
```

---

## 六、CSRF 防护

**CSRF 攻击原理：** 诱骗已认证用户在不知情的情况下提交恶意请求（利用浏览器自动携带 Cookie 的特性）。

**Spring Security 防护机制（同步器令牌模式）：**

1. 服务器为每个会话生成唯一 CSRF 令牌，存入 Session
2. 表单提交时附带 CSRF 令牌（隐藏域或请求头）
3. 服务器比较请求中的令牌和 Session 中的令牌

CSRF 防护默认启用。**JWT 无状态 API 可以禁用**，因为 JWT 手动放在请求头，浏览器不会自动发送。

---

## 七、XSS 防护

**Spring Security 提供的 HTTP 安全响应头：**

```java
http.headers(headers -> headers
    .contentSecurityPolicy(csp -> csp.policyDirectives("default-src 'self'"))  // 核心防御
    .frameOptions(frame -> frame.sameOrigin())  // 防点击劫持
    .xssProtection(xss -> xss.block(true))
);
```

**根本防御：** 输出编码——将用户数据渲染给浏览器前，根据上下文进行正确的 HTML/JS/URL 编码，Thymeleaf 等模板引擎默认自动 HTML 转义。

---

## 八、接口签名验证

接口签名用于开放 API 场景，确保请求完整性和来源真实性，防止中间人篡改。

**流程：**

1. **客户端生成签名**：所有请求参数按字典序排序拼接 + secretKey → HMAC-SHA256 哈希
2. **服务端验签（拦截器实现）**：
   - 检查时间戳（防重放攻击，窗口一般 5 分钟）
   - 检查 Nonce 是否已使用（Redis 短期缓存）
   - 按同样规则生成签名，比对客户端签名

---

## 九、密码加密

**BCrypt 是密码存储的黄金标准，** 核心特性：

- **自动生成随机盐**：相同密码哈希结果完全不同，防彩虹表攻击
- **工作因子可调**：成本每增加 1，计算时间约翻倍，随硬件发展调整保持强度
- **输出格式**：包含算法版本、成本因子、盐和哈希，60 位字符串

```java
@Bean
public PasswordEncoder passwordEncoder() {
    return new BCryptPasswordEncoder();  // 默认强度 10，生产可用 12
}

// 存储密码
passwordEncoder.encode(rawPassword)

// 验证密码（Spring Security 自动调用，无需手动）
passwordEncoder.matches(rawPassword, encodedPassword)
```

---

## 十、单点登录（SSO）

SSO 基于 OAuth 2.0 / OpenID Connect 协议实现，核心是一个中央认证服务器（如 Keycloak、Spring Authorization Server）。

**流程：** 用户访问应用 A → 重定向到认证中心登录 → 回调携带授权码 → 换取令牌 → 建立本地会话。用户访问应用 B 时，认证中心检测到已登录，直接重定向回应用 B，无需再次输入密码。

---

## 十一、动态权限管理

动态权限是指权限规则（URL-角色映射）存储在数据库，可在不重启应用的情况下实时修改生效。

**实现核心：**

1. 数据库存储权限规则（URL pattern、HTTP method、required role）
2. 实现 `FilterInvocationSecurityMetadataSource`，从数据库动态加载所需角色
3. 实现 `AccessDecisionManager`，根据当前用户权限和所需权限做放行决策
4. 通过 `withObjectPostProcessor` 注入到 `FilterSecurityInterceptor`
5. 权限规则加 Redis 缓存，管理员修改后清缓存

---

## 关联面试题

- 📝 [[10_Developlanguage/005_Spring/03_SpringBootSubject/06、安全与认证]]
