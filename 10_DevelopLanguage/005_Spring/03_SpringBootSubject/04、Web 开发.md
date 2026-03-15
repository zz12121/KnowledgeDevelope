###### 1. Spring Boot 如何集成 Spring MVC？

引入 `spring-boot-starter-web` 依赖，Spring Boot 的 `WebMvcAutoConfiguration` 就会自动完成所有基础配置：注册 `DispatcherServlet`（映射到 `/`）、配置静态资源映射、注册 `HttpMessageConverter`（Jackson 序列化 JSON）、配置 `ViewResolver`（如果有 Thymeleaf 就自动配上）。

扩展 MVC 配置时，实现 `WebMvcConfigurer` 接口重写需要的方法即可。**注意不要加 `@EnableWebMvc`**——这个注解会完全接管 MVC 配置，把 Spring Boot 的自动配置全部关掉。

📖 [[26_SpringBootKnowledge/03_Web开发/01、SpringBoot Web开发]]

---

###### 2. @RestController 和 @Controller 的区别是什么？

`@Controller` 是标准 MVC 控制器，方法默认返回视图名，由 `ViewResolver` 渲染成页面，适合服务端渲染的传统 Web 应用。

`@RestController` 是 `@Controller` + `@ResponseBody` 的组合注解，类里所有方法默认都有 `@ResponseBody` 语义，返回值直接通过 `HttpMessageConverter` 序列化成 JSON/XML 写到响应体，适合前后端分离的 RESTful API。

用 `@Controller` 开发接口时，需要在每个方法上单独加 `@ResponseBody`；用 `@RestController` 就省了这步，整个类都是返回数据的。

📖 [[26_SpringBootKnowledge/03_Web开发/01、SpringBoot Web开发]]

---

###### 3. 如何处理跨域请求（CORS）？

跨域问题由浏览器同源策略引发，Spring Boot 提供三种解决方案：

**1. `@CrossOrigin` 注解**：局部配置，加在类或方法上，适合个别接口的特殊跨域需求。

**2. `WebMvcConfigurer.addCorsMappings()`（推荐）**：全局配置，统一管理，避免到处写注解：

```java
@Override
public void addCorsMappings(CorsRegistry registry) {
    registry.addMapping("/api/**")
            .allowedOrigins("https://frontend-app.com")
            .allowedMethods("GET", "POST", "PUT", "DELETE")
            .allowCredentials(true);
}
```

**3. `CorsFilter`**：在 Filter 层处理跨域，执行比 DispatcherServlet 还早，适合需要配合 Spring Security 使用的场景（Spring Security 的过滤器链需要在最前面看到 CORS 响应头）。

📖 [[26_SpringBootKnowledge/03_Web开发/01、SpringBoot Web开发]]

---

###### 4. Spring Boot 如何实现文件上传？

Spring Boot 通过 `MultipartAutoConfiguration` 自动配置，只需在配置文件里设置文件大小限制：

```yaml
spring:
  servlet:
    multipart:
      max-file-size: 10MB
      max-request-size: 100MB
```

Controller 里用 `MultipartFile` 接收上传的文件：

```java
@PostMapping("/upload")
public ResponseEntity<String> upload(@RequestParam("file") MultipartFile file) {
    if (file.isEmpty()) return ResponseEntity.badRequest().body("请选择文件");
    
    String fileName = System.currentTimeMillis() + "_" + file.getOriginalFilename();
    Path target = Paths.get("/upload/dir", fileName);
    Files.createDirectories(target.getParent());
    file.transferTo(target.toFile());
    
    return ResponseEntity.ok("上传成功: " + fileName);
}
```

多文件上传用 `MultipartFile[]` 数组接收。实际生产中文件一般不存本地，而是上传到 OSS（阿里云/AWS S3）。

📖 [[26_SpringBootKnowledge/03_Web开发/01、SpringBoot Web开发]]

---

###### 5. 如何自定义错误页面？

**简单场景**：在 `src/main/resources/static/error/` 目录下放 `404.html`、`500.html`，Spring Boot 的 `BasicErrorController` 会自动匹配并返回。

**复杂场景**：实现 `ErrorController` 接口，自定义错误处理逻辑：

```java
@Controller
@RequestMapping("/error")
public class MyErrorController implements ErrorController {

    @RequestMapping
    public String handleError(HttpServletRequest request, Model model) {
        Integer statusCode = (Integer) request.getAttribute("javax.servlet.error.status_code");
        if (statusCode == 404) return "error/404";
        model.addAttribute("statusCode", statusCode);
        return "error/generic";
    }
}
```

实际项目里 REST API 通常不用错误页面，而是在全局异常处理器里统一返回 JSON 格式的错误响应。

📖 [[26_SpringBootKnowledge/03_Web开发/01、SpringBoot Web开发]]

---

###### 6. 如何实现全局异常处理？

用 `@RestControllerAdvice` 注解（`@ControllerAdvice` + `@ResponseBody` 的组合），在里面用 `@ExceptionHandler` 处理各类异常：

```java
@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(BusinessException.class)
    public ResponseEntity<ApiResponse<?>> handleBusiness(BusinessException e) {
        return ResponseEntity.badRequest()
                .body(ApiResponse.error(e.getCode(), e.getMessage()));
    }

    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ResponseEntity<ApiResponse<?>> handleValidation(MethodArgumentNotValidException e) {
        String errors = e.getBindingResult().getFieldErrors().stream()
                .map(fe -> fe.getField() + ": " + fe.getDefaultMessage())
                .collect(Collectors.joining(", "));
        return ResponseEntity.badRequest().body(ApiResponse.error("VALIDATION_ERROR", errors));
    }

    @ExceptionHandler(Exception.class) // 兜底，处理所有未被捕获的异常
    public ResponseEntity<ApiResponse<?>> handleAll(Exception e) {
        log.error("未处理异常: ", e);
        return ResponseEntity.status(500).body(ApiResponse.error("SYSTEM_ERROR", "系统繁忙"));
    }
}
```

全局异常处理器让所有的异常处理逻辑集中在一处，Controller 里只需关注正常业务逻辑，不用到处写 try-catch。

📖 [[26_SpringBootKnowledge/03_Web开发/01、SpringBoot Web开发]]

---

###### 7. @ControllerAdvice 的作用是什么？

`@ControllerAdvice` 是一个**全局增强注解**，被它标注的类会对所有 `@Controller` 类生效（也可以通过 `basePackages` 属性限定范围）。

它支持三类增强：
1. `@ExceptionHandler`：全局异常处理
2. `@InitBinder`：全局数据绑定初始化（比如统一的日期格式转换）
3. `@ModelAttribute`：全局模型数据预填充

`@RestControllerAdvice` 是 `@ControllerAdvice` + `@ResponseBody` 的组合，在 REST API 项目里用这个更方便，所有方法的返回值都直接序列化到响应体。

📖 [[26_SpringBootKnowledge/03_Web开发/01、SpringBoot Web开发]]

---

###### 8. 如何实现参数校验？

Spring Boot 内置了 Bean Validation 支持（`spring-boot-starter-validation`），三步搞定参数校验：

**第一步：在 DTO 上加校验注解**
```java
public class UserDTO {
    @NotBlank(message = "用户名不能为空")
    @Size(min = 2, max = 10)
    private String username;

    @Email
    private String email;

    @Min(value = 18)
    private Integer age;
}
```

**第二步：在 Controller 参数上加 `@Valid`**
```java
@PostMapping("/users")
public ResponseEntity<?> create(@Valid @RequestBody UserDTO dto) {
    // 校验失败会自动抛出 MethodArgumentNotValidException
}
```

**第三步：在全局异常处理器里处理校验异常**（见上题）

自定义校验注解也很简单，实现 `ConstraintValidator` 接口就行，比如手机号格式校验、枚举值校验等。

📖 [[26_SpringBootKnowledge/03_Web开发/01、SpringBoot Web开发]]

---

###### 9. Spring Boot 如何集成模板引擎（Thymeleaf/FreeMarker）？

Spring Boot 对主流模板引擎都有 Starter 支持，引入依赖就自动配置好了。

**Thymeleaf**（推荐，Spring 官方偏好）：

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-thymeleaf</artifactId>
</dependency>
```

自动配置默认模板路径为 `classpath:/templates/`，后缀为 `.html`。开发阶段建议关闭缓存：`spring.thymeleaf.cache=false`。

**FreeMarker**：引入 `spring-boot-starter-freemarker`，模板后缀默认 `.ftl`，配置 `spring.freemarker.cache=false` 关闭缓存。

两者都是服务端模板引擎，现代项目前后端分离的越来越多，如果你做的是纯 API 服务，这两个都不需要。

📖 [[26_SpringBootKnowledge/03_Web开发/01、SpringBoot Web开发]]

---

###### 10. 如何配置静态资源访问路径？

Spring Boot 默认映射 `/**` 到以下目录（优先级由高到低）：
1. `classpath:/META-INF/resources/`
2. `classpath:/resources/`
3. `classpath:/static/`
4. `classpath:/public/`

自定义路径或添加磁盘目录：

```java
@Override
public void addResourceHandlers(ResourceHandlerRegistry registry) {
    registry.addResourceHandler("/assets/**")
            .addResourceLocations("classpath:/assets/", "file:/opt/static/")
            .setCacheControl(CacheControl.maxAge(1, TimeUnit.HOURS));
}
```

生产环境静态资源要配置 HTTP 强缓存（`Cache-Control: max-age`），减少重复请求，提升性能。

📖 [[26_SpringBootKnowledge/03_Web开发/01、SpringBoot Web开发]]

---

###### 11. Spring Boot 如何实现拦截器？

实现 `HandlerInterceptor` 接口，然后通过 `WebMvcConfigurer.addInterceptors()` 注册：

```java
// 实现
public class AuthInterceptor implements HandlerInterceptor {
    @Override
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) {
        String token = request.getHeader("Authorization");
        if (!isValid(token)) {
            response.setStatus(401);
            return false; // 返回 false 中断后续处理
        }
        return true;
    }
}

// 注册
@Configuration
public class WebConfig implements WebMvcConfigurer {
    @Override
    public void addInterceptors(InterceptorRegistry registry) {
        registry.addInterceptor(new AuthInterceptor())
                .addPathPatterns("/api/**")
                .excludePathPatterns("/api/public/**");
    }
}
```

Spring Boot 中拦截器可以直接 `@Autowired` 注入 Spring Bean，比过滤器方便得多。

📖 [[26_SpringBootKnowledge/03_Web开发/01、SpringBoot Web开发]]

---

###### 12. 拦截器和过滤器的区别是什么？

核心区别在于**执行位置**和**能力范围**：

过滤器（Filter）是 Servlet 规范的组件，在 DispatcherServlet 之前执行，可以拦截所有 Web 请求（包括静态资源），但无法访问 Spring MVC 的 HandlerMethod、ModelAndView 等对象。

拦截器（Interceptor）是 Spring MVC 的组件，在 DispatcherServlet 之后执行，只拦截映射到 Controller 的请求，但可以访问 Spring 上下文，支持 `@Autowired` 注入。

执行顺序：`Filter → DispatcherServlet → Interceptor → Controller`

**选择原则**：需要在 Spring MVC 处理之前执行（比如 Token 过滤）或需要拦截静态资源，用 Filter；需要访问 Spring 组件、操作 Controller 执行结果，用 Interceptor。

📖 [[26_SpringBootKnowledge/03_Web开发/01、SpringBoot Web开发]]

---

###### 13. 如何实现请求日志记录？

最推荐的方式是 AOP + 自定义注解，按需给需要记录日志的接口打注解：

```java
@Aspect
@Component
@Slf4j
public class RequestLogAspect {

    @Around("@annotation(loggable)")
    public Object logRequest(ProceedingJoinPoint joinPoint, Loggable loggable) throws Throwable {
        long start = System.currentTimeMillis();
        HttpServletRequest request = getCurrentRequest();

        log.info("请求开始: {} {}", request.getMethod(), request.getRequestURI());
        try {
            Object result = joinPoint.proceed();
            log.info("请求完成: {} {}, 耗时: {}ms", request.getMethod(),
                    request.getRequestURI(), System.currentTimeMillis() - start);
            return result;
        } catch (Exception ex) {
            log.error("请求异常: {} {}, 耗时: {}ms",
                    request.getMethod(), request.getRequestURI(),
                    System.currentTimeMillis() - start, ex);
            throw ex;
        }
    }
}
```

也可以用拦截器实现全量接口日志，或者引入 Spring Boot 的 `CommonsRequestLoggingFilter` 自动记录请求体和响应体。

📖 [[26_SpringBootKnowledge/03_Web开发/01、SpringBoot Web开发]]

---

###### 14. Spring Boot 如何实现接口版本控制？

最简单也最常用的是 **URL 路径版本控制**，直接在 `@RequestMapping` 里加版本号：

```java
@RestController
@RequestMapping("/api/v1/users")
public class UserControllerV1 { ... }

@RestController
@RequestMapping("/api/v2/users")
public class UserControllerV2 { ... }
```

如果需要更灵活的版本路由（比如向下兼容，v2 没有的接口自动走 v1），可以自定义 `RequestMappingHandlerMapping`，实现版本继承逻辑。

另外还有基于请求头（`Accept: application/vnd.app.v1+json`）和查询参数（`?version=1`）的版本控制，但 URL 版本控制最直观，最常见。

📖 [[26_SpringBootKnowledge/03_Web开发/01、SpringBoot Web开发]]

---

###### 15. 如何实现 RESTful API 的最佳实践？

核心原则：

**1. 资源命名用名词复数**：`/users`、`/orders`，用 HTTP 方法（GET/POST/PUT/DELETE）表达操作，不要在 URL 里写动词（不要 `/getUser`、`/deleteUser`）。

**2. 合理使用 HTTP 状态码**：
- `200 OK`：成功
- `201 Created`：创建成功，响应头带 `Location` 指向新资源
- `400 Bad Request`：参数错误
- `401 Unauthorized`：未认证
- `403 Forbidden`：没权限
- `404 Not Found`：资源不存在
- `500 Internal Server Error`：服务端错误

**3. 统一响应格式**：用统一的 `ApiResponse<T>` 包装，包含 code、message、data、timestamp，让前端有统一的处理规范。

**4. 版本化 API**：在路径里加版本号（`/api/v1/`），便于后续迭代。

**5. 用 `ResponseEntity` 精确控制响应**：

```java
@PostMapping("/users")
public ResponseEntity<User> create(@Valid @RequestBody UserDTO dto) {
    User user = userService.create(dto);
    URI location = ServletUriComponentsBuilder.fromCurrentRequest()
            .path("/{id}").buildAndExpand(user.getId()).toUri();
    return ResponseEntity.created(location).body(user); // 201 + Location 头
}
```

📖 [[26_SpringBootKnowledge/03_Web开发/01、SpringBoot Web开发]]

---

###### 16. Spring Boot 如何实现接口限流？

常用方案是 **Guava RateLimiter + AOP**，简单高效：

```java
@Aspect
@Component
public class RateLimitAspect {
    private final ConcurrentHashMap<String, RateLimiter> limiters = new ConcurrentHashMap<>();

    @Around("@annotation(rateLimit)")
    public Object limit(ProceedingJoinPoint joinPoint, RateLimit rateLimit) throws Throwable {
        String key = joinPoint.getSignature().toString();
        RateLimiter limiter = limiters.computeIfAbsent(key,
            k -> RateLimiter.create(rateLimit.permitsPerSecond()));

        if (limiter.tryAcquire(500, TimeUnit.MILLISECONDS)) {
            return joinPoint.proceed();
        }
        throw new RateLimitException("请求过于频繁，请稍后重试");
    }
}
```

分布式场景（多实例部署）需要用 Redis 实现分布式限流，可以用 Redis + Lua 脚本的滑动窗口算法，或者直接接入 Sentinel 限流框架。

📖 [[26_SpringBootKnowledge/03_Web开发/01、SpringBoot Web开发]]

---

###### 17. 如何实现接口幂等性？

参考 SpringMVC 章节的幂等性实现，核心思路是 **Token + Redis SETNX**：

客户端请求前先获取一个唯一 Token，携带 Token 提交请求，服务端用 SETNX 检查 Token 是否存在：首次请求设置成功，正常处理；重复请求设置失败，直接拒绝。

```java
@Override
public boolean preHandle(HttpServletRequest request, ...) {
    String token = request.getHeader("Idempotent-Token");
    Boolean success = redisTemplate.opsForValue()
        .setIfAbsent("idempotent:" + token, "1", Duration.ofSeconds(3600));
    if (!Boolean.TRUE.equals(success)) {
        throw new IdempotentException("请勿重复提交");
    }
    return true;
}
```

📖 [[26_SpringBootKnowledge/03_Web开发/01、SpringBoot Web开发]]

---

###### 18. Spring Boot 如何集成 Swagger/OpenAPI？

Spring Boot 3.x 推荐使用 `springdoc-openapi`：

```xml
<dependency>
    <groupId>org.springdoc</groupId>
    <artifactId>springdoc-openapi-starter-webmvc-ui</artifactId>
    <version>2.3.0</version>
</dependency>
```

引入后访问 `http://localhost:8080/swagger-ui.html` 就能看到接口文档，无需额外配置。

在 Controller 上用注解补充说明：

```java
@RestController
@Tag(name = "用户管理")
@RequestMapping("/api/users")
public class UserController {

    @Operation(summary = "获取用户详情")
    @GetMapping("/{id}")
    public ResponseEntity<User> getUser(
            @Parameter(description = "用户ID") @PathVariable Long id) { ... }
}
```

生产环境记得关掉 Swagger 接口文档（`springdoc.swagger-ui.enabled=false`），避免泄露接口信息。

📖 [[26_SpringBootKnowledge/03_Web开发/01、SpringBoot Web开发]]
