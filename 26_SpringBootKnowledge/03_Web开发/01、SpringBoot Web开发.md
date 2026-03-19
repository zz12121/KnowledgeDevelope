# SpringBoot Web 开发

## Spring Boot 集成 Spring MVC

引入 `spring-boot-starter-web` 后，`WebMvcAutoConfiguration` 自动配置类负责完成所有 MVC 基础设置：

- 自动注册 `DispatcherServlet`，映射到 `/` 路径
- 配置静态资源映射（`/static`、`/public` 等目录直接可访问）
- 注册默认 `ViewResolver`（如果有 Thymeleaf 就配 Thymeleaf）
- 注册 `HttpMessageConverter`（Jackson 序列化 JSON）

如果需要扩展 MVC 配置，实现 `WebMvcConfigurer` 接口并重写对应方法即可，**不要加 `@EnableWebMvc`**——加了会完全接管 MVC 配置，导致 Spring Boot 的自动配置失效。

---

## @RestController vs @Controller

- `@Controller`：标准 MVC 控制器，方法返回视图名，交给 `ViewResolver` 渲染页面，用于服务端渲染场景
- `@RestController`：`@Controller` + `@ResponseBody` 的组合，方法返回值直接序列化到响应体，用于 RESTful API 场景

```java
// 传统 MVC：返回视图名
@Controller
public class PageController {
    @GetMapping("/user/page")
    public String userPage(Model model) {
        return "user/detail"; // 渲染 templates/user/detail.html
    }
}

// RESTful API：返回数据
@RestController
public class ApiController {
    @GetMapping("/api/users/{id}")
    public User getUser(@PathVariable Long id) {
        return userService.findById(id); // 自动序列化为 JSON
    }
}
```

---

## 跨域处理（CORS）

三种配置方式，从细到粗：

**1. 局部注解 `@CrossOrigin`**（适合个别接口）：

```java
@CrossOrigin(origins = "https://trusted-domain.com")
@GetMapping("/public/data")
public String getData() { ... }
```

**2. 全局配置（推荐）**：

```java
@Configuration
public class CorsConfig implements WebMvcConfigurer {
    @Override
    public void addCorsMappings(CorsRegistry registry) {
        registry.addMapping("/api/**")
                .allowedOrigins("https://frontend-app.com")
                .allowedMethods("GET", "POST", "PUT", "DELETE")
                .allowCredentials(true);
    }
}
```

**3. `CorsFilter`**（适合需要在 Filter 层处理跨域，比如配合 Spring Security）：

```java
@Bean
public CorsFilter corsFilter() {
    CorsConfiguration config = new CorsConfiguration();
    config.setAllowedOrigins(List.of("https://frontend-app.com"));
    config.addAllowedMethod("*");
    UrlBasedCorsConfigurationSource source = new UrlBasedCorsConfigurationSource();
    source.registerCorsConfiguration("/**", config);
    return new CorsFilter(source);
}
```

---

## 文件上传

Spring Boot 通过 `MultipartAutoConfiguration` 自动配置，只需配置文件大小限制就能用：

```yaml
spring:
  servlet:
    multipart:
      max-file-size: 10MB        # 单文件上限
      max-request-size: 100MB    # 整个请求上限
```

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

---

## 全局异常处理

`@RestControllerAdvice` 是 `@ControllerAdvice` + `@ResponseBody` 的组合，专为 REST API 的全局异常处理而生：

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
        List<String> errors = e.getBindingResult().getFieldErrors().stream()
                .map(fe -> fe.getField() + ": " + fe.getDefaultMessage())
                .collect(Collectors.toList());
        return ResponseEntity.badRequest()
                .body(ApiResponse.error("VALIDATION_ERROR", errors.toString()));
    }

    @ExceptionHandler(Exception.class)
    public ResponseEntity<ApiResponse<?>> handleAll(Exception e) {
        log.error("未处理异常: ", e);
        return ResponseEntity.status(500)
                .body(ApiResponse.error("SYSTEM_ERROR", "系统繁忙，请稍后重试"));
    }
}
```

---

## 参数校验

引入 `spring-boot-starter-validation`，在 DTO 上加校验注解，在 Controller 方法参数上加 `@Valid`：

```java
public class UserDTO {
    @NotBlank(message = "用户名不能为空")
    @Size(min = 2, max = 10)
    private String username;

    @Email(message = "邮箱格式不正确")
    private String email;

    @Min(value = 18, message = "年龄须大于18")
    private Integer age;
}

@PostMapping("/users")
public ResponseEntity<?> createUser(@Valid @RequestBody UserDTO dto) {
    // 校验失败自动抛出 MethodArgumentNotValidException
    // 被全局异常处理器捕获，不需要在这里处理
    userService.save(dto);
    return ResponseEntity.ok("创建成功");
}
```

---

## 静态资源配置

默认静态资源目录（优先级由高到低）：
1. `classpath:/META-INF/resources/`
2. `classpath:/resources/`
3. `classpath:/static/`
4. `classpath:/public/`

自定义映射：

```java
@Override
public void addResourceHandlers(ResourceHandlerRegistry registry) {
    registry.addResourceHandler("/assets/**")
            .addResourceLocations("classpath:/assets/", "file:/opt/static/")
            .setCacheControl(CacheControl.maxAge(1, TimeUnit.HOURS));
}
```

---

## 拦截器

```java
// 1. 实现 HandlerInterceptor
public class AuthInterceptor implements HandlerInterceptor {
    @Override
    public boolean preHandle(HttpServletRequest request,
                             HttpServletResponse response, Object handler) {
        String token = request.getHeader("Authorization");
        if (!isValid(token)) {
            response.setStatus(401);
            return false; // 中断请求
        }
        return true;
    }
}

// 2. 注册
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

拦截器 vs 过滤器的核心区别：拦截器是 Spring MVC 的组件，只拦截 Controller 请求，可以访问 Spring 上下文；过滤器是 Servlet 规范的组件，拦截所有 Web 请求（包括静态资源），执行更早。

---

## 接口文档（Swagger/OpenAPI）

Spring Boot 3.x 推荐用 `springdoc-openapi`：

```xml
<dependency>
    <groupId>org.springdoc</groupId>
    <artifactId>springdoc-openapi-starter-webmvc-ui</artifactId>
    <version>2.3.0</version>
</dependency>
```

启动后访问 `http://localhost:8080/swagger-ui.html` 即可查看接口文档。在 Controller 上用 `@Tag`、`@Operation`、`@Parameter` 等注解补充说明。

---

## PDF 补充内容

### 1. 静态资源访问原理解析

静态资源访问流程：
1. 请求进来，先去找 Controller 看能不能处理
2. 如果不能处理，则尝试去寻找静态资源
3. 可以通过 `【当前项目根路径 + 静态资源名】` 即 `http://localhost:8080/kangxi.jpg` 的方式访问静态资源

**相关源码**位于 `WebMvcAutoConfiguration.addResourceHandlers()` 方法中：

```java
private String staticPathPattern = "/**";
private String[] staticLocations = CLASSPATH_RESOURCE_LOCATIONS;
private static final String[] CLASSPATH_RESOURCE_LOCATIONS = { 
    "classpath:/META-INF/resources/",
    "classpath:/resources/", 
    "classpath:/static/", 
    "classpath:/public/" 
};
```

**默认静态资源目录优先级**（由高到低）：
1. `classpath:/META-INF/resources/`
2. `classpath:/resources/`
3. `classpath:/static/`
4. `classpath:/public/`

### 2. Rest 风格请求映射

Rest 风格支持使用 **HTTP 请求方式动词** 来表示对资源的操作：

| HTTP 方法 | 操作 | 示例 |
|-----------|------|------|
| GET | 查询 | GET /users |
| POST | 新增 | POST /users |
| PUT | 完整更新 | PUT /users/1 |
| PATCH | 部分更新 | PATCH /users/1 |
| DELETE | 删除 | DELETE /users/1 |

**携带 `_method` 的表单提交：**

HTML 表单只支持 GET 和 POST，需要通过隐藏字段 `_method` 指定实际请求方式：

```html
<form action="/users/1" method="post">
    <input type="hidden" name="_method" value="delete">
</form>
```

需要在 `application.yml` 中开启 HiddenHttpMethodFilter：

```yaml
spring:
  mvc:
    hiddenmethod:
      filter:
        enabled: true
```

### 3. HiddenHttpMethodFilter 源码解析

`HiddenHttpMethodFilter` 是实现 Rest 风格的核心 Filter：

```java
public class HiddenHttpMethodFilter extends OncePerRequestFilter {
    
    private static final String METHOD_PARAMETER = "_method";
    
    @Override
    protected void doFilterInternal(HttpServletRequest request,
            HttpServletResponse response, FilterChain filterChain)
            throws ServletException, IOException {
        
        HttpServletRequest requestToUse = request;
        
        // 判断是否为 POST 请求，且包含 _method 参数
        if ("POST".equals(request.getMethod()) && request.getParameter(METHOD_PARAMETER) != null) {
            String method = request.getParameter(METHOD_PARAMETER).toUpperCase(Locale.ENGLISH);
            // 只支持指定的方法类型
            if (ALLOWED_METHODS.contains(method)) {
                // 包装请求，将实际方法改为指定的方法
                requestToUse = new HttpMethodRequestWrapper(request, method);
            }
        }
        
        filterChain.doFilter(requestToUse, response);
    }
    
    private static final Set<String> ALLOWED_METHODS = 
            EnumSet.of("PUT", "DELETE", "PATCH");
}
```

### 4. 自定义入参 Converter 实现

如果需要将请求参数转换为特定的 Java 对象，可以自定义 `Converter`：

```java
@Configuration
public class WebConfig implements WebMvcConfigurer {
    
    @Override
    public void addFormatters(FormatterRegistry registry) {
        // 注册自定义 Converter
        registry.addConverter(new StringToUserConverter());
    }
}

public class StringToUserConverter implements Converter<String, User> {
    @Override
    public User convert(String source) {
        // source 格式：id,name
        String[] parts = source.split(",");
        User user = new User();
        user.setId(Long.parseLong(parts[0]));
        user.setName(parts[1]);
        return user;
    }
}
```

使用示例：
```java
@GetMapping("/user")
public User getUser(User user) {
    // 请求 /user?id=1,name=test 会自动将参数转换为 User 对象
    return user;
}
```

---

## 相关面试题 →

[[../../10_Developlanguage/005_Spring/03_SpringBootSubject/04、Web 开发]]
