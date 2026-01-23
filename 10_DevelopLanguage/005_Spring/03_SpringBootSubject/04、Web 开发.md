###### 1. Spring Boot 如何集成 Spring MVC?
Spring Boot通过**自动配置机制**无缝集成Spring MVC，核心在于`spring-boot-starter-web`依赖和`WebMvcAutoConfiguration`自动化配置类。
**自动配置核心流程：**
1. **依赖引入**：添加`spring-boot-starter-web`会传递引入Spring MVC核心模块、嵌入式Tomcat及JSON处理库（如Jackson）。
2. **条件化配置生效**：Spring Boot应用启动时，`WebMvcAutoConfiguration`类在检测到类路径下存在Servlet API、Spring MVC等资源后自动激活。其核心职责包括：
    - 自动注册`DispatcherServlet`，并映射到`/`路径，替代传统`web.xml`配置。
    - 配置默认的`ViewResolver`（如`InternalResourceViewResolver`用于JSP，`ThymeleafViewResolver`用于Thymeleaf）。
    - 配置静态资源映射（`/static`、`/public`等目录可直接访问）。
    - 注册`HttpMessageConverter`用于JSON/XML序列化。
**扩展配置**：若需覆盖默认配置，可实现`WebMvcConfigurer`接口并重写相关方法（如添加拦截器、格式化器），或使用`@EnableWebMvc`注解（此注解会完全接管MVC配置，需谨慎使用）。
###### 2. @RestController 和 @Controller 的区别是什么?
两者均用于定义控制器，但设计目标和行为有本质差异。

|**特性**​|**@Controller**​|**@RestController**​|
|---|---|---|
|**核心语义**​|标记类为传统MVC控制器，侧重**页面渲染**。|专为RESTful API设计，是`@Controller`和`@ResponseBody`的**组合注解**，侧重**数据返回**。|
|**返回值处理**​|方法返回值通常为**视图名（String）**，由`ViewResolver`解析为物理视图（如JSP、HTML）。|方法返回值直接**通过`HttpMessageConverter`序列化**（如JSON/XML）写入HTTP响应体，不进行视图解析。|
|**适用场景**​|前后端未分离的应用，需要服务端渲染页面。|前后端分离的架构，提供纯数据接口。|
|**源码差异**​|标准Spring MVC注解，无特殊组合。|元注解定义包含`@Controller`和`@ResponseBody`，其语义由此组合实现。|
**代码示例对比：**
```java
// 使用 @Controller，需要配合 @ResponseBody 返回JSON
@Controller
public class TraditionalController {
    @GetMapping("/user/page")
    public String userPage(Model model) {
        // 返回视图名
        return "userInfo";
    }
    
    @ResponseBody
    @GetMapping("/api/user")
    public User getUserData() {
        // 手动添加@ResponseBody使返回数据直接写入body
        return userService.findUser(1L);
    }
}

// 使用 @RestController，所有方法默认具有 @ResponseBody 语义
@RestController
public class ApiController {
    @GetMapping("/api/user")
    public User getUser() {
        // 返回值自动序列化为JSON
        return userService.findUser(1L);
    }
}
```
###### 3. 如何处理跨域请求（CORS）?
跨域问题由浏览器同源策略引发，Spring Boot提供三种主流解决方案。
1. **局部配置 - `@CrossOrigin`注解**：
    在控制器类或方法上直接使用，配置灵活但粒度较细。
    ```java
    @RestController
    @CrossOrigin(origins = "https://trusted-domain.com", maxAge = 3600)
    public class ApiController {
        @CrossOrigin(origins = "*") // 方法级配置覆盖类级配置
        @GetMapping("/public/data")
        public String getPublicData() {
            return "Public Data";
        }
    }
    ```
2. **全局配置 - 实现`WebMvcConfigurer`接口**（推荐）：
    通过重写`addCorsMappings`方法统一管理，避免重复配置。
```java
    @Configuration
    public class CorsConfig implements WebMvcConfigurer {
        @Override
        public void addCorsMappings(CorsRegistry registry) {
            registry.addMapping("/api/**") // 匹配API路径
                    .allowedOrigins("https://frontend-app.com") // 允许的源
                    .allowedMethods("GET", "POST", "PUT", "DELETE") // 允许的HTTP方法
                    .allowedHeaders("*") // 允许的请求头
                    .allowCredentials(true); // 是否允许发送Cookie
        }
    }
```
3. **过滤器方案 - 自定义`CorsFilter`**：
    ```java
    @Bean
    public CorsFilter corsFilter() {
        UrlBasedCorsConfigurationSource source = new UrlBasedCorsConfigurationSource();
        CorsConfiguration config = new CorsConfiguration();
        config.applyPermitDefaultValues();
        config.setAllowedOrigins(Arrays.asList("https://frontend-app.com"));
        source.registerCorsConfiguration("/**", config);
        return new CorsFilter(source);
    }
    ```
###### 4. Spring Boot 如何实现文件上传?
Spring Boot通过`MultipartAutoConfiguration`自动配置`MultipartResolver`，简化文件上传。
**核心步骤：**
1. **配置文件上传属性**（`application.properties`）：
    ```properties
    # 设置单个文件最大大小
    spring.servlet.multipart.max-file-size=10MB
    # 设置整个请求最大大小
    spring.servlet.multipart.max-request-size=100MB
    # 指定文件上传的临时目录
    spring.servlet.multipart.location=/tmp/uploads
    ```
2. **控制器处理上传请求**：
    使用`@RequestParam`接收`MultipartFile`对象，它封装了文件内容及元数据。
	```java
    @RestController
    public class FileUploadController {
        @PostMapping("/upload")
        public ResponseEntity<String> handleFileUpload(@RequestParam("file") MultipartFile file) {
            if (file.isEmpty()) {
                return ResponseEntity.badRequest().body("Please select a file.");
            }
            try {
                // 生成唯一文件名避免冲突
                String fileName = System.currentTimeMillis() + "_" + file.getOriginalFilename();
                Path filePath = Paths.get("/upload/dir", fileName);
    
                // 确保目录存在并保存文件
                Files.createDirectories(filePath.getParent());
                file.transferTo(filePath.toFile());
    
                return ResponseEntity.ok("File uploaded successfully: " + fileName);
            } catch (IOException e) {
                return ResponseEntity.status(HttpStatus.INTERNAL_SERVER_ERROR).body("Upload failed.");
            }
        }
    
        // 多文件上传
        @PostMapping("/uploadMultiple")
        public ResponseEntity<String> handleMultipleFiles(@RequestParam("files") MultipartFile[] files) {
            // 遍历files数组处理每个文件
            Arrays.stream(files).forEach(this::saveFile);
            return ResponseEntity.ok("All files uploaded.");
        }
    }
	```
###### 5. 如何自定义错误页面?
Spring Boot提供两种方式自定义错误页面，提升用户体验。
1. **静态错误页面**（简单场景）：
    在`src/main/resources/static/error/`或`src/main/resources/templates/error/`目录下创建对应HTTP状态码的页面，如`404.html`、`5xx.html`。Spring Boot的`BasicErrorController`会自动匹配并返回相应页面。
2. **动态错误页面（推荐）**：
    通过自定义`ErrorController`实现更灵活的错误处理，可记录日志、跳转特定页面。
    ```java
    @Controller
    @RequestMapping("/error") // 映射到Spring Boot默认的错误路径
    public class MyCustomErrorController implements ErrorController {
    
        @RequestMapping
        public String handleError(HttpServletRequest request, Model model) {
            // 从请求属性中获取错误信息
            Integer statusCode = (Integer) request.getAttribute("javax.servlet.error.status_code");
            Exception exception = (Exception) request.getAttribute("javax.servlet.error.exception");
    
            // 记录日志
            if (exception != null) {
                logger.error("Error occurred: ", exception);
            }
    
            // 根据状态码跳转到不同视图
            if (statusCode == 404) {
                return "error/404";
            } else if (statusCode == 403) {
                return "error/403";
            } else {
                model.addAttribute("statusCode", statusCode);
                return "error/generic";
            }
        }
    }
    ```
###### 6. 如何实现全局异常处理?
`@ControllerAdvice`是Spring MVC中实现**全局异常处理**的核心注解，它允许开发者在一个地方集中处理整个Web控制层抛出的异常，避免在每个Controller中重复编写异常处理代码。
###### 7. @ControllerAdvice 的作用是什么?
**`@ControllerAdvice`的工作原理**：
标记了`@ControllerAdvice`的类是一个**全局拦截器**，它会对所有被`@Controller`或`@RestController`注解的控制器方法进行拦截。当控制器方法抛出异常时，如果该异常未被控制器内部的`@ExceptionHandler`处理，就会被`@ControllerAdvice`类中匹配的`@ExceptionHandler`方法捕获。
**全局异常处理实践：**
```java
// @RestControllerAdvice 是 @ControllerAdvice 和 @ResponseBody 的组合注解，专用于REST API
@RestControllerAdvice(basePackages = "com.example.controller") // 可指定包范围
public class GlobalExceptionHandler {
    
    private static final Logger logger = LoggerFactory.getLogger(GlobalExceptionHandler.class);
    
    /**
     * 处理业务异常
     */
    @ExceptionHandler(BusinessException.class)
    public ResponseEntity<ApiResponse<?>> handleBusinessException(BusinessException e) {
        logger.warn("Business exception occurred: ", e);
        ApiResponse<?> errorResponse = ApiResponse.error(e.getErrorCode(), e.getMessage());
        return ResponseEntity.status(HttpStatus.BAD_REQUEST).body(errorResponse);
    }
    
    /**
     * 处理数据校验异常（@Validated 触发的）
     */
    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ResponseEntity<ApiResponse<?>> handleValidationException(MethodArgumentNotValidException e) {
        List<String> errors = e.getBindingResult().getFieldErrors().stream()
                .map(error -> error.getField() + ": " + error.getDefaultMessage())
                .collect(Collectors.toList());
        ApiResponse<?> errorResponse = ApiResponse.error("VALIDATION_FAILED", "数据校验失败", errors);
        return ResponseEntity.status(HttpStatus.BAD_REQUEST).body(errorResponse);
    }
    
    /**
     * 处理所有未显式处理的异常
     */
    @ExceptionHandler(Exception.class)
    public ResponseEntity<ApiResponse<?>> handleGlobalException(Exception e) {
        logger.error("Unexpected exception occurred: ", e);
        ApiResponse<?> errorResponse = ApiResponse.error("INTERNAL_ERROR", "系统繁忙，请稍后重试");
        return ResponseEntity.status(HttpStatus.INTERNAL_SERVER_ERROR).body(errorResponse);
    }
}

// 统一的API响应体
public class ApiResponse<T> {
    private String code;
    private String message;
    private T data;
    
    // 构造方法、静态工厂方法等
    public static <T> ApiResponse<T> success(T data) {
        ApiResponse<T> response = new ApiResponse<>();
        response.setCode("SUCCESS");
        response.setMessage("操作成功");
        response.setData(data);
        return response;
    }
    
    public static <T> ApiResponse<T> error(String code, String message) {
        ApiResponse<T> response = new ApiResponse<>();
        response.setCode(code);
        response.setMessage(message);
        return response;
    }
}
```
###### 8. 如何实现参数校验?
Spring Boot通过集成**Bean Validation API**（Hibernate Validator为其实现）实现参数校验。
**实现步骤：**
1. **添加依赖**（`spring-boot-starter-validation`）：
    ```xml
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-validation</artifactId>
    </dependency>
    ```
2. **在实体类/DTO上定义校验规则**：
 ```java
    public class UserDTO {
        @NotNull(message = "用户ID不能为空")
        private Long id;
    
        @NotBlank(message = "用户名不能为空")
        @Size(min = 2, max = 10, message = "用户名长度必须在2-10个字符之间")
        private String username;
    
        @Email(message = "邮箱格式不正确")
        private String email;
    
        @Min(value = 18, message = "年龄必须大于18岁")
        private Integer age;
    
        // 自定义校验注解
        @PhoneNumber(message = "手机号格式不正确")
        private String phone;
    
        // getters and setters
    }
 ```
1. **在Controller方法中使用`@Validated`或`@Valid`触发校验**：
    ```java
    @RestController
    @Validated // 类级别启用校验（支持方法参数校验）
    public class UserController {
    
        // 校验请求体
        @PostMapping("/users")
        public ResponseEntity<?> createUser(@Valid @RequestBody UserDTO userDTO) {
            // 如果校验失败，会抛出MethodArgumentNotValidException，由全局异常处理器捕获
            userService.save(userDTO);
            return ResponseEntity.ok("用户创建成功");
        }
    
        // 校验路径变量和查询参数
        @GetMapping("/users/{id}")
        public UserDTO getUserById(@PathVariable @Min(1) Long id, 
                                  @RequestParam @NotBlank String type) {
            return userService.findById(id);
        }
    }
    ```
1. **自定义校验注解**（高级用法）：
    ```java
    @Target({ElementType.FIELD})
    @Retention(RetentionPolicy.RUNTIME)
    @Constraint(validatedBy = PhoneNumberValidator.class)
    public @interface PhoneNumber {
        String message() default "手机号格式不正确";
        Class<?>[] groups() default {};
        Class<? extends Payload>[] payload() default {};
    }
    
    public class PhoneNumberValidator implements ConstraintValidator<PhoneNumber, String> {
        private static final Pattern PHONE_PATTERN = Pattern.compile("^1[3-9]\\d{9}$");
    
        @Override
        public boolean isValid(String value, ConstraintValidatorContext context) {
            if (value == null) return true; // 结合@NotNull使用
            return PHONE_PATTERN.matcher(value).matches();
        }
    }
    ```
###### 9. Spring Boot 如何集成模板引擎（Thymeleaf/FreeMarker）?
Spring Boot通过自动配置机制简化了模板引擎的集成，主要支持Thymeleaf和FreeMarker两种主流模板引擎。
Thymeleaf集成详解
**依赖配置**：
```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-thymeleaf</artifactId>
</dependency>
```
**自动配置原理**：
Spring Boot的`ThymeleafAutoConfiguration`类在检测到类路径存在Thymeleaf相关类时自动生效。关键配置属性通过`ThymeleafProperties`类绑定：
```java
@ConfigurationProperties(prefix = "spring.thymeleaf")
public class ThymeleafProperties {
    private static final Charset DEFAULT_ENCODING = StandardCharsets.UTF_8;
    public static final String DEFAULT_PREFIX = "classpath:/templates/";
    public static final String DEFAULT_SUFFIX = ".html";
    private boolean cache = true; // 生产环境建议开启
}
```
**配置示例**：
```yaml
spring:
  thymeleaf:
    prefix: classpath:/templates/
    suffix: .html
    mode: HTML
    encoding: UTF-8
    cache: false # 开发环境关闭缓存
    servlet:
      content-type: text/html
```
**控制器与模板交互**：
```java
@Controller
public class UserController {
    
    @GetMapping("/user/{id}")
    public String userProfile(@PathVariable Long id, Model model) {
        User user = userService.findById(id);
        model.addAttribute("user", user);
        model.addAttribute("currentTime", LocalDateTime.now());
        return "user/profile"; // 对应templates/user/profile.html
    }
}
```
FreeMarker集成详解
**依赖配置**：
```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-freemarker</artifactId>
</dependency>
```
**配置属性**：
```properties
spring.freemarker.template-loader-path=classpath:/templates/
spring.freemarker.suffix=.ftl
spring.freemarker.cache=false
spring.freemarker.charset=UTF-8
spring.freemarker.content-type=text/html
```
**FreeMarker模板特性**：
- 支持宏定义和嵌套
- 强大的字符串、列表、Map操作能力
- 日期和数字格式化支持
###### 10. 如何配置静态资源访问路径?
Spring Boot提供了默认的静态资源映射规则，同时也支持自定义配置。
默认静态资源映射
Spring Boot自动将`/**`映射到以下目录（按优先级排序）：
1. `classpath:/META-INF/resources/`
2. `classpath:/resources/`
3. `classpath:/static/`
4. `classpath:/public/`
**源码分析**：
```java
// ResourceProperties 中定义了默认位置
public class ResourceProperties {
    private static final String[] CLASSPATH_RESOURCE_LOCATIONS = {
        "classpath:/META-INF/resources/",
        "classpath:/resources/", 
        "classpath:/static/",
        "classpath:/public/"
    };
}
```
自定义静态资源配置
**编程式配置**：
```java
@Configuration
public class WebMvcConfig implements WebMvcConfigurer {
    
    @Override
    public void addResourceHandlers(ResourceHandlerRegistry registry) {
        // 自定义目录映射
        registry.addResourceHandler("/assets/**")
                .addResourceLocations("classpath:/assets/", "file:/opt/static/")
                .setCachePeriod(3600)
                .setCacheControl(CacheControl.maxAge(1, TimeUnit.HOURS));
        
        // 保留默认配置
        registry.addResourceHandler("/**")
                .addResourceLocations(
                    "classpath:/META-INF/resources/",
                    "classpath:/resources/",
                    "classpath:/static/",
                    "classpath:/public/"
                );
    }
}
```
**配置文件中配置**：
```properties
# 自定义静态资源路径
spring.mvc.static-path-pattern=/resources/**
spring.resources.static-locations=classpath:/custom-static/
```
###### 11. Spring Boot 如何实现拦截器?
拦截器基于AOP思想，在请求处理前后插入自定义逻辑。
拦截器实现步骤
**1. 实现HandlerInterceptor接口**：
```java
@Slf4j
public class AuthInterceptor implements HandlerInterceptor {
    
    @Override
    public boolean preHandle(HttpServletRequest request, 
                           HttpServletResponse response, Object handler) {
        log.info("请求进入拦截器: {}", request.getRequestURI());
        
        // 身份验证逻辑
        String token = request.getHeader("Authorization");
        if (!isValidToken(token)) {
            sendErrorResponse(response, 401, "未授权访问");
            return false; // 中断请求
        }
        return true;
    }
    
    @Override
    public void postHandle(HttpServletRequest request, HttpServletResponse response,
                         Object handler, ModelAndView modelAndView) {
        log.info("控制器执行完成: {}", request.getRequestURI());
    }
    
    @Override
    public void afterCompletion(HttpServletRequest request, 
                              HttpServletResponse response, Object handler, Exception ex) {
        log.info("请求处理完成: {}", request.getRequestURI());
        if (ex != null) {
            log.error("请求处理异常", ex);
        }
    }
}
```
**2. 注册拦截器**：
```java
@Configuration
public class InterceptorConfig implements WebMvcConfigurer {
    
    @Override
    public void addInterceptors(InterceptorRegistry registry) {
        registry.addInterceptor(new AuthInterceptor())
                .addPathPatterns("/api/**") // 拦截路径
                .excludePathPatterns("/api/public/**"); // 排除路径
    }
}
```
###### 12. 拦截器和过滤器的区别是什么?

|**特性**​|**拦截器(Interceptor)**​|**过滤器(Filter)**​|
|---|---|---|
|**规范依赖**​|Spring框架特定组件|Servlet规范标准组件|
|**作用范围**​|仅针对Spring MVC控制器|所有Web请求（静态资源、Servlet等）|
|**依赖注入**​|支持Spring依赖注入|不支持，需通过其他方式获取Bean|
|**实现机制**​|基于Java反射和动态代理|基于函数回调|
|**控制粒度**​|方法级别拦截|URL模式匹配|
|**访问上下文**​|可访问HandlerMethod、ModelAndView等Spring对象|仅能访问Servlet API|
**执行顺序**：Filter → DispatcherServlet → Interceptor → Controller
###### 13. 如何实现请求日志记录?
AOP方式实现请求日志记录
**定义日志注解**：
```java
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
public @interface Loggable {
    String value() default "";
    boolean recordParams() default true;
    boolean recordResult() default false;
}
```
**AOP切面实现**：
```java
@Aspect
@Component
@Slf4j
public class RequestLogAspect {
    
    @Around("@annotation(loggable)")
    public Object logRequest(ProceedingJoinPoint joinPoint, Loggable loggable) throws Throwable {
        long startTime = System.currentTimeMillis();
        HttpServletRequest request = getCurrentRequest();
        
        // 记录请求信息
        log.info("请求开始: {} {}, 参数: {}", 
                request.getMethod(), request.getRequestURI(), getParams(joinPoint));
        
        try {
            Object result = joinPoint.proceed();
            long cost = System.currentTimeMillis() - startTime;
            
            // 记录响应信息
            if (loggable.recordResult()) {
                log.info("请求完成: {} {}, 耗时: {}ms, 结果: {}", 
                        request.getMethod(), request.getRequestURI(), cost, result);
            } else {
                log.info("请求完成: {} {}, 耗时: {}ms", 
                        request.getMethod(), request.getRequestURI(), cost);
            }
            return result;
        } catch (Exception ex) {
            long cost = System.currentTimeMillis() - startTime;
            log.error("请求异常: {} {}, 耗时: {}ms, 异常: {}", 
                    request.getMethod(), request.getRequestURI(), cost, ex.getMessage(), ex);
            throw ex;
        }
    }
}
```
###### 14. Spring Boot 如何实现接口版本控制?
基于URL路径的版本控制
**配置路由策略**：
```java
@Configuration
public class ApiVersionConfig implements WebMvcConfigurer {
    
    @Override
    public void configurePathMatch(PathMatchConfigurer configurer) {
        UrlPathHelper urlPathHelper = new UrlPathHelper();
        urlPathHelper.setRemoveSemicolonContent(false);
        configurer.setUrlPathHelper(urlPathHelper);
    }
}
```
**自定义版本注解**：
```java
@Target({ElementType.TYPE, ElementType.METHOD})
@Retention(RetentionPolicy.RUNTIME)
@RequestMapping
public @interface ApiVersion {
    String value();
}
```
**版本路由解析器**：
```java
public class ApiVersionHandlerMapping extends RequestMappingHandlerMapping {
    
    @Override
    protected RequestCondition<?> getCustomTypeCondition(Class<?> handlerType) {
        ApiVersion typeAnnotation = handlerType.getAnnotation(ApiVersion.class);
        return typeAnnotation != null ? new ApiVersionCondition(typeAnnotation.value()) : null;
    }
    
    @Override
    protected RequestCondition<?> getCustomMethodCondition(Method method) {
        ApiVersion methodAnnotation = method.getAnnotation(ApiVersion.class);
        return methodAnnotation != null ? new ApiVersionCondition(methodAnnotation.value()) : null;
    }
}
```
**控制器使用示例**：
```java
@RestController
@ApiVersion("1")
@RequestMapping("/api/v{version}/users")
public class UserControllerV1 {
    
    @GetMapping("/{id}")
    public UserV1 getUserV1(@PathVariable Long id) {
        // V1版本实现
    }
}

@RestController
@ApiVersion("2")  
@RequestMapping("/api/v{version}/users")
public class UserControllerV2 {
    
    @GetMapping("/{id}")
    public UserV2 getUserV2(@PathVariable Long id) {
        // V2版本实现
    }
}
```
###### 15. 如何实现 RESTful API 的最佳实践?
设计原则与规范
**1. 资源命名规范**
- 使用名词复数形式：`/users`、`/orders`
- 层次化资源：`/users/{userId}/orders`
- 避免动词，使用HTTP方法表达操作
**2. HTTP状态码规范**
```java
@RestController
public class UserController {
    
    @GetMapping("/users/{id}")
    public ResponseEntity<User> getUser(@PathVariable Long id) {
        User user = userService.findById(id);
        if (user == null) {
            return ResponseEntity.notFound().build();
        }
        return ResponseEntity.ok(user);
    }
    
    @PostMapping("/users")
    public ResponseEntity<User> createUser(@Valid @RequestBody User user) {
        User savedUser = userService.save(user);
        URI location = ServletUriComponentsBuilder.fromCurrentRequest()
                .path("/{id}").buildAndExpand(savedUser.getId()).toUri();
        return ResponseEntity.created(location).body(savedUser);
    }
}
```
**3. 统一响应格式**
```java
public class ApiResponse<T> {
    private String code;
    private String message;
    private T data;
    private long timestamp;
    
    // 静态工厂方法
    public static <T> ApiResponse<T> success(T data) {
        return new ApiResponse<>("SUCCESS", "操作成功", data, System.currentTimeMillis());
    }
}
```
###### 16. Spring Boot 如何实现接口限流?
基于Guava RateLimiter的限流方案
**自定义限流注解**：
```java
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
public @interface RateLimit {
    String key() default "";
    double permitsPerSecond(); // 每秒允许的请求数
    int timeout() default 500; // 获取许可的超时时间(ms)
}
```
**限流切面实现**：
```java
@Aspect
@Component
public class RateLimitAspect {
    
    private final ConcurrentHashMap<String, RateLimiter> limiters = new ConcurrentHashMap<>();
    
    @Around("@annotation(rateLimit)")
    public Object rateLimit(ProceedingJoinPoint joinPoint, RateLimit rateLimit) throws Throwable {
        String key = generateKey(joinPoint, rateLimit);
        RateLimiter limiter = limiters.computeIfAbsent(key, 
            k -> RateLimiter.create(rateLimit.permitsPerSecond()));
        
        if (limiter.tryAcquire(rateLimit.timeout(), TimeUnit.MILLISECONDS)) {
            return joinPoint.proceed();
        } else {
            throw new RateLimitException("请求过于频繁，请稍后重试");
        }
    }
}
```
###### 17. 如何实现接口幂等性?
基于Token机制的幂等性实现
**幂等性注解**：
```java
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
public @interface Idempotent {
    String key() default "";
    long expireTime() default 3600; // 令牌过期时间(秒)
}
```
**幂等性校验拦截器**：
```java
@Component
public class IdempotentInterceptor implements HandlerInterceptor {
    
    @Autowired
    private RedisTemplate<String, Object> redisTemplate;
    
    @Override
    public boolean preHandle(HttpServletRequest request, 
                           HttpServletResponse response, Object handler) {
        if (isIdempotentRequired(handler)) {
            String token = request.getHeader("Idempotent-Token");
            if (!StringUtils.hasText(token)) {
                throw new IdempotentException("缺少幂等令牌");
            }
            
            String key = "idempotent:" + token;
            Boolean absent = redisTemplate.opsForValue()
                .setIfAbsent(key, "processing", Duration.ofSeconds(getExpireTime(handler)));
                
            if (Boolean.FALSE.equals(absent)) {
                throw new IdempotentException("请勿重复提交");
            }
        }
        return true;
    }
}
```
###### 18. Spring Boot 如何集成 Swagger/OpenAPI?
SpringDoc OpenAPI 3集成
**依赖配置**：
```xml
<dependency>
    <groupId>org.springdoc</groupId>
    <artifactId>springdoc-openapi-starter-webmvc-ui</artifactId>
    <version>2.3.0</version>
</dependency>
```
**配置类**：
```java
@Configuration
@OpenAPIDefinition(info = @Info(
    title = "API文档",
    version = "1.0",
    description = "系统API文档"
))
public class OpenApiConfig {
    
    @Bean
    public OpenAPI customOpenAPI() {
        return new OpenAPI()
                .components(new Components()
                        .addSecuritySchemes("bearer-key",
                                new SecurityScheme().type(SecurityScheme.Type.HTTP)
                                        .scheme("bearer").bearerFormat("JWT")))
                .info(new Info().title("API文档").version("1.0"));
    }
}
```
**控制器文档注解**：
```java
@RestController
@Tag(name = "用户管理", description = "用户相关API")
@RequestMapping("/api/users")
public class UserController {
    
    @Operation(summary = "获取用户详情", description = "根据用户ID获取用户详细信息")
    @ApiResponses({
        @ApiResponse(responseCode = "200", description = "成功"),
        @ApiResponse(responseCode = "404", description = "用户不存在")
    })
    @GetMapping("/{id}")
    public ResponseEntity<User> getUser(
            @Parameter(description = "用户ID") @PathVariable Long id) {
        // 实现逻辑
    }
}
```