# Spring Security 过滤器链原理

## 一、整体架构

Spring Security 的本质是一条 **Servlet Filter 责任链**，所有 HTTP 请求在到达业务 Controller 之前，必须依次通过该链中的每一个 Filter 的检查。

```
客户端请求
  └── Servlet 容器（Tomcat）
       └── DelegatingFilterProxy（Servlet Filter，Spring 的桥接层）
            └── FilterChainProxy（Spring Security 的总入口）
                 └── SecurityFilterChain（具体的过滤器链）
                      ├── SecurityContextPersistenceFilter
                      ├── UsernamePasswordAuthenticationFilter
                      ├── AnonymousAuthenticationFilter
                      ├── ExceptionTranslationFilter
                      └── FilterSecurityInterceptor（最终授权决策者）
                           └── Spring MVC DispatcherServlet → Controller
```

---

## 二、核心组件详解

### 2.1 DelegatingFilterProxy

`DelegatingFilterProxy` 是 Servlet 规范的 `Filter` 实现，它本身不做任何安全逻辑。其核心作用是**桥接 Servlet 容器与 Spring 容器**：

```java
public class DelegatingFilterProxy extends GenericFilterBean {
    // 懒加载：第一次请求时才从 Spring 容器获取 FilterChainProxy
    @Override
    public void doFilter(ServletRequest request, ServletResponse response, FilterChain filterChain) {
        // 延迟加载：第一次调用时从 Spring ApplicationContext 中获取 delegate
        Filter delegateToUse = this.delegate;
        if (delegateToUse == null) {
            // targetBeanName 默认是 "springSecurityFilterChain"
            delegateToUse = initDelegate(getWebApplicationContext());
        }
        // 委托给 FilterChainProxy 处理
        delegateToUse.doFilter(request, response, filterChain);
    }
}
```

### 2.2 FilterChainProxy（Spring Security 总入口）

`FilterChainProxy` 是真正的 Spring Security 入口，它管理多条 `SecurityFilterChain`，根据请求 URL 选择匹配的过滤器链：

```java
public class FilterChainProxy extends GenericFilterBean {

    // 可以配置多条 SecurityFilterChain（每个 HttpSecurity 配置块对应一条）
    private List<SecurityFilterChain> filterChains;

    @Override
    public void doFilter(ServletRequest request, ServletResponse response, FilterChain chain) {
        // 核心方法：选择匹配的 SecurityFilterChain
        doFilterInternal(request, response, chain);
    }

    private void doFilterInternal(HttpServletRequest request, ...) {
        // 找到第一条匹配的 SecurityFilterChain（按配置顺序匹配）
        List<Filter> filters = getFilters(request);
        if (filters == null || filters.isEmpty()) {
            // 没有匹配的过滤器链，直接放行
            chain.doFilter(request, response);
            return;
        }
        // 生成 VirtualFilterChain，执行所有 Security Filter
        VirtualFilterChain vfc = new VirtualFilterChain(fwRequest, chain, filters);
        vfc.doFilter(fwRequest, response);
    }

    private static final class VirtualFilterChain implements FilterChain {
        private int currentPosition = 0;
        private final List<Filter> additionalFilters;

        @Override
        public void doFilter(ServletRequest request, ServletResponse response) {
            if (this.currentPosition == this.size) {
                // 所有 Security Filter 执行完毕，继续后续的 Servlet 处理
                this.originalChain.doFilter(request, response);
            } else {
                this.currentPosition++;
                Filter nextFilter = this.additionalFilters.get(this.currentPosition - 1);
                nextFilter.doFilter(request, response, this);
            }
        }
    }
}
```

---

## 三、关键过滤器源码分析

### 3.1 SecurityContextPersistenceFilter（安全上下文持久化）

请求开始时从存储（默认 HttpSession）加载 `SecurityContext`，请求结束后保存回存储，确保跨请求保持登录状态：

```java
public class SecurityContextPersistenceFilter extends GenericFilterBean {

    // 默认使用 HttpSessionSecurityContextRepository，从 Session 中读写
    private SecurityContextRepository repo;

    public void doFilter(ServletRequest req, ...) {
        // 1. 从 Session（或其他存储）加载 SecurityContext
        SecurityContext contextBeforeChainExecution = repo.loadContext(holder);
        // 2. 将 SecurityContext 放入线程本地变量（ThreadLocal）
        SecurityContextHolder.setContext(contextBeforeChainExecution);
        try {
            chain.doFilter(request, response);
        } finally {
            // 3. 请求结束时，将更新后的 SecurityContext 保存回 Session
            SecurityContext contextAfterChainExecution = SecurityContextHolder.getContext();
            SecurityContextHolder.clearContext();  // 清除 ThreadLocal（防止内存泄漏）
            repo.saveContext(contextAfterChainExecution, holder.getRequest(), holder.getResponse());
        }
    }
}
```

### 3.2 UsernamePasswordAuthenticationFilter（用户名密码认证）

处理 `POST /login` 请求，提取用户名密码，委托 `AuthenticationManager` 完成认证：

```java
public class UsernamePasswordAuthenticationFilter extends AbstractAuthenticationProcessingFilter {

    @Override
    public Authentication attemptAuthentication(HttpServletRequest request, ...) {
        // 只处理 POST 请求
        if (!request.getMethod().equals("POST")) {
            throw new AuthenticationServiceException("...");
        }
        // 从请求中提取用户名和密码
        String username = obtainUsername(request);  // 从 "username" 参数
        String password = obtainPassword(request);  // 从 "password" 参数

        // 封装成 UsernamePasswordAuthenticationToken（尚未认证）
        UsernamePasswordAuthenticationToken authRequest =
            new UsernamePasswordAuthenticationToken(username, password);

        // 委托 AuthenticationManager 进行认证
        return this.getAuthenticationManager().authenticate(authRequest);
    }
}
```

**AbstractAuthenticationProcessingFilter（父类）中的成功/失败处理**：
```java
private void doFilter(HttpServletRequest request, ...) {
    if (!requiresAuthentication(request, response)) {
        // 不是登录请求，直接放行
        chain.doFilter(request, response);
        return;
    }
    try {
        // 调用子类 attemptAuthentication()
        Authentication authenticationResult = attemptAuthentication(request, response);
        // 认证成功：将 Authentication 存入 SecurityContext，重定向到成功页
        successfulAuthentication(request, response, chain, authenticationResult);
    } catch (AuthenticationException ex) {
        // 认证失败：清除 SecurityContext，重定向到失败页
        unsuccessfulAuthentication(request, response, ex);
    }
}
```

### 3.3 AnonymousAuthenticationFilter（匿名身份）

如果当前请求没有已认证的 Authentication，为其设置一个匿名 Authentication，确保 `SecurityContextHolder` 始终不为 null：

```java
public class AnonymousAuthenticationFilter extends GenericFilterBean {
    @Override
    public void doFilter(ServletRequest req, ...) {
        if (SecurityContextHolder.getContext().getAuthentication() == null) {
            // 未认证的请求，设置匿名 Authentication
            SecurityContextHolder.getContext().setAuthentication(
                createAuthentication((HttpServletRequest) req)
                // AnonymousAuthenticationToken(key, "anonymousUser", ROLE_ANONYMOUS)
            );
        }
        chain.doFilter(req, response);
    }
}
```

### 3.4 ExceptionTranslationFilter（异常翻译）

捕获 `FilterSecurityInterceptor` 抛出的两类异常，转化为 HTTP 响应：

```java
public class ExceptionTranslationFilter extends GenericFilterBean {

    @Override
    public void doFilter(ServletRequest req, ...) {
        try {
            chain.doFilter(request, response);
        } catch (AuthenticationException ex) {
            // 认证异常（未登录）：触发 AuthenticationEntryPoint
            // 默认行为：重定向到 /login
            handleSpringSecurityException(request, response, chain, ex);
        } catch (AccessDeniedException ex) {
            // 授权异常（无权限）：
            Authentication authentication = SecurityContextHolder.getContext().getAuthentication();
            if (authenticationTrustResolver.isAnonymous(authentication)) {
                // 是匿名用户 → 重定向到登录页（AuthenticationEntryPoint）
                handleSpringSecurityException(request, response, chain, new InsufficientAuthenticationException(...));
            } else {
                // 已登录但无权限 → 返回 403（AccessDeniedHandler）
                accessDeniedHandler.handle(request, response, ex);
            }
        }
    }
}
```

### 3.5 FilterSecurityInterceptor（最终授权决策）

过滤器链的最后一道关卡，决定当前请求是否有访问资源的权限：

```java
public class FilterSecurityInterceptor extends AbstractSecurityInterceptor implements Filter {

    @Override
    public void doFilter(ServletRequest request, ...) {
        // 委托父类 AbstractSecurityInterceptor 处理
        FilterInvocation fi = new FilterInvocation(request, response, chain);
        invoke(fi);
    }

    public void invoke(FilterInvocation filterInvocation) {
        // 1. 调用父类的 beforeInvocation() 进行权限检查
        InterceptorStatusToken token = super.beforeInvocation(filterInvocation);
        // 2. 通过则执行实际的 Servlet（Controller）
        filterInvocation.getChain().doFilter(filterInvocation.getRequest(), ...);
    }
}

// AbstractSecurityInterceptor.beforeInvocation() 核心逻辑：
protected InterceptorStatusToken beforeInvocation(Object object) {
    // 获取该请求需要的权限配置（如 hasRole('ADMIN')）
    Collection<ConfigAttribute> attributes = this.obtainSecurityMetadataSource().getAttributes(object);

    // 获取当前用户的 Authentication
    Authentication authenticated = SecurityContextHolder.getContext().getAuthentication();

    // 委托 AccessDecisionManager 进行决策
    // 默认是 AffirmativeBased（一票通过），调用所有 AccessDecisionVoter
    this.accessDecisionManager.decide(authenticated, object, attributes);

    return new InterceptorStatusToken(...);
}
```

---

## 四、认证核心：AuthenticationManager 与 AuthenticationProvider

```
authenticate(token)
  └── AuthenticationManager（接口）
       └── ProviderManager（最常用实现）
            ├── 遍历所有 AuthenticationProvider
            └── DaoAuthenticationProvider（默认，基于 UserDetailsService）
                 ├── userDetailsService.loadUserByUsername(username)  ← 开发者实现这里
                 ├── 比较密码（使用 PasswordEncoder）
                 └── 返回 UsernamePasswordAuthenticationToken（已认证，含 authorities）
```

```java
public class DaoAuthenticationProvider extends AbstractUserDetailsAuthenticationProvider {

    @Override
    protected void additionalAuthenticationChecks(UserDetails userDetails,
            UsernamePasswordAuthenticationToken authentication) {
        // 密码校验
        if (!passwordEncoder.matches(
                authentication.getCredentials().toString(), userDetails.getPassword())) {
            throw new BadCredentialsException("Bad credentials");
        }
    }

    @Override
    protected UserDetails retrieveUser(String username, ...) {
        // 调用开发者实现的 UserDetailsService.loadUserByUsername()
        UserDetails loadedUser = this.getUserDetailsService().loadUserByUsername(username);
        if (loadedUser == null) {
            throw new InternalAuthenticationServiceException("UserDetailsService returned null");
        }
        return loadedUser;
    }
}
```

---

## 五、完整认证流程总结

```
POST /login (username=xxx, password=xxx)
  ├── SecurityContextPersistenceFilter → 加载 SecurityContext（首次为空）
  ├── UsernamePasswordAuthenticationFilter
  │    ├── requiresAuthentication() → true（POST /login）
  │    ├── attemptAuthentication()
  │    │    └── AuthenticationManager.authenticate(UsernamePasswordAuthenticationToken)
  │    │         └── ProviderManager → DaoAuthenticationProvider
  │    │              ├── UserDetailsService.loadUserByUsername(username)  ← 查数据库
  │    │              └── PasswordEncoder.matches(rawPwd, encodedPwd)
  │    ├── 认证成功 → successfulAuthentication()
  │    │    ├── SecurityContextHolder.setContext(authenticated)
  │    │    └── 重定向到 "/"（successUrl）
  │    └── 认证失败 → unsuccessfulAuthentication()
  │         └── 重定向到 "/login?error"
  └── SecurityContextPersistenceFilter → 将认证成功的 SecurityContext 存入 Session
```

```
GET /protected-resource（已登录用户）
  ├── SecurityContextPersistenceFilter → 从 Session 恢复 SecurityContext（含 Authentication）
  ├── UsernamePasswordAuthenticationFilter → requiresAuthentication() = false，放行
  ├── AnonymousAuthenticationFilter → 已有 Authentication，跳过
  ├── ExceptionTranslationFilter → 包裹后续执行，捕获异常
  └── FilterSecurityInterceptor
       ├── 获取当前请求需要的权限（如 ROLE_ADMIN）
       ├── AccessDecisionManager.decide(authentication, request, configAttributes)
       │    └── 认证通过 → 继续；权限不足 → 抛 AccessDeniedException
       └── chain.doFilter → DispatcherServlet → Controller
```

---

## 六、SecurityContextHolder 与 ThreadLocal

`SecurityContextHolder` 使用 `ThreadLocal` 存储当前请求的 `SecurityContext`，使认证信息在整个请求处理线程中随时可取：

```java
// 在 Controller 中直接获取当前登录用户
Authentication authentication = SecurityContextHolder.getContext().getAuthentication();
String username = authentication.getName();
Collection<? extends GrantedAuthority> authorities = authentication.getAuthorities();
```

**注意事项**：
- 在异步执行（`@Async`、新线程）中，`SecurityContext` 默认不会自动传播
- 可通过 `SecurityContextHolder.setStrategyName(SecurityContextHolder.MODE_INHERITABLETHREADLOCAL)` 改用 `InheritableThreadLocal`，使子线程继承父线程的安全上下文

---

## 相关面试题

- [[../10_DevelopLanguage/005_Spring/04_SpringCloudSubject/10、安全认证|📖 10、安全认证]]
