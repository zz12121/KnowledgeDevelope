# RESTful 设计与文件处理

## RESTful 风格

REST（Representational State Transfer，表述性状态转移）是 Roy Fielding 2000 年提出的软件架构风格，把互联网看作由资源构成的分布式系统，每个资源通过 URI 唯一标识，通过 HTTP 标准方法操作。

**与 RPC 风格的对比：**
- RPC 风格：关注方法调用，URL 是动词（`/getUser`、`/createOrder`、`/deleteUser`）
- REST 风格：关注资源状态，URL 是名词，动作由 HTTP 方法表达（`GET /users/123`、`POST /orders`、`DELETE /users/123`）

---

## HTTP 方法语义

| HTTP 方法 | 语义 | 幂等性 | 常见返回码 |
|-----------|------|--------|----------|
| `GET` | 查询资源 | 幂等 | 200 OK |
| `POST` | 创建资源 | 非幂等 | 201 Created |
| `PUT` | 完整更新（替换整个资源） | 幂等 | 200 OK |
| `PATCH` | 部分更新（只更新提供的字段） | 非幂等 | 200 OK |
| `DELETE` | 删除资源 | 幂等 | 204 No Content |

**RESTful URL 设计原则：**
- 使用名词（复数）：`/users`、`/orders`
- 体现层级关系：`/users/{userId}/orders/{orderId}`
- API 版本化：`/api/v1/users`
- 查询/过滤/分页用查询参数：`/users?status=active&page=1&size=20`

---

## RESTful Controller 示例

```java
@RestController
@RequestMapping("/api/v1/users")
public class UserController {
    
    @GetMapping
    public PageResult<User> list(@RequestParam(defaultValue = "1") int page,
                                 @RequestParam(defaultValue = "20") int size) {
        return userService.findPage(page, size);
    }
    
    @GetMapping("/{id}")
    public ResponseEntity<User> getById(@PathVariable Long id) {
        return userService.findById(id)
            .map(ResponseEntity::ok)
            .orElse(ResponseEntity.notFound().build());
    }
    
    @PostMapping
    public ResponseEntity<User> create(@Valid @RequestBody UserCreateDTO dto) {
        User user = userService.create(dto);
        URI location = URI.create("/api/v1/users/" + user.getId());
        return ResponseEntity.created(location).body(user);
    }
    
    @PutMapping("/{id}")
    public ResponseEntity<User> update(@PathVariable Long id,
                                       @Valid @RequestBody UserUpdateDTO dto) {
        return ResponseEntity.ok(userService.update(id, dto));
    }
    
    @DeleteMapping("/{id}")
    public ResponseEntity<Void> delete(@PathVariable Long id) {
        userService.delete(id);
        return ResponseEntity.noContent().build();
    }
}
```

`ResponseEntity` 是构建 REST 响应的利器，提供对状态码、响应头、响应体的精细控制。

---

## 跨域处理（CORS）

同源策略（协议+域名+端口完全相同）限制了浏览器的跨域请求。SpringMVC 提供三种处理方式：

**1. `@CrossOrigin` 注解**（细粒度，控制单个接口）：
```java
@CrossOrigin(origins = "https://frontend.example.com", 
             allowedHeaders = "*",
             methods = {RequestMethod.GET, RequestMethod.POST},
             maxAge = 3600)
@RestController
public class UserController { ... }
```

**2. 全局配置**（推荐，统一管理）：
```java
@Configuration
public class WebConfig implements WebMvcConfigurer {
    @Override
    public void addCorsMappings(CorsRegistry registry) {
        registry.addMapping("/api/**")
                .allowedOrigins("https://frontend.example.com")
                .allowedMethods("GET", "POST", "PUT", "DELETE")
                .allowedHeaders("*")
                .allowCredentials(true)
                .maxAge(3600);
    }
}
```

**3. `CorsFilter`**（Servlet 层面，优先级最高）：适合需要在 Spring Security 之前处理跨域的场景。

---

## 文件上传

SpringMVC 通过 `MultipartResolver` 处理文件上传请求（`Content-Type: multipart/form-data`）：

**配置**（Spring Boot 中默认已启用）：
```yaml
spring:
  servlet:
    multipart:
      max-file-size: 10MB      # 单文件大小限制
      max-request-size: 50MB   # 请求总大小限制
      file-size-threshold: 1MB # 超过此大小写临时文件，否则在内存中
```

**处理文件上传**：
```java
@PostMapping("/upload")
public Result upload(@RequestParam("file") MultipartFile file,
                     @RequestParam(required = false) String description) throws IOException {
    if (file.isEmpty()) return Result.fail("请选择文件");
    
    String originalName = file.getOriginalFilename();
    String extension = StringUtils.getFilenameExtension(originalName);
    String newName = UUID.randomUUID() + "." + extension;
    
    // 大文件用流，小文件用 getBytes()
    Path targetPath = Paths.get(uploadDir, newName);
    file.transferTo(targetPath);  // 最高效的方式，直接转存
    
    return Result.ok(Map.of("url", "/files/" + newName));
}

// 多文件上传
@PostMapping("/uploadMultiple")
public Result uploadMultiple(@RequestParam("files") List<MultipartFile> files) {
    List<String> urls = files.stream()
        .map(this::saveFile)
        .collect(Collectors.toList());
    return Result.ok(urls);
}
```

---

## 文件下载

正确设置 HTTP 响应头是文件下载的关键，推荐用 `ResponseEntity<Resource>`：

```java
@GetMapping("/download/{filename}")
public ResponseEntity<Resource> download(@PathVariable String filename) throws IOException {
    Path filePath = Paths.get(uploadDir, filename);
    Resource resource = new UrlResource(filePath.toUri());
    
    if (!resource.exists() || !resource.isReadable()) {
        throw new BusinessException(404, "文件不存在");
    }
    
    String contentType = Files.probeContentType(filePath);
    return ResponseEntity.ok()
            .contentType(MediaType.parseMediaType(contentType))
            .header(HttpHeaders.CONTENT_DISPOSITION,
                "attachment; filename=\"" + URLEncoder.encode(filename, "UTF-8") + "\"")
            .body(resource);
}
```

`Content-Disposition: attachment` 告诉浏览器下载文件而不是直接打开；`inline` 则直接在浏览器展示（如 PDF 预览）。

---

## 静态资源处理

DispatcherServlet 映射到 `/` 时会拦截所有请求，包括静态资源。解决方案：

```java
@Configuration
public class WebConfig implements WebMvcConfigurer {
    @Override
    public void addResourceHandlers(ResourceHandlerRegistry registry) {
        // 映射静态资源目录
        registry.addResourceHandler("/static/**")
                .addResourceLocations("classpath:/static/", "file:/opt/uploads/")
                .setCacheControl(CacheControl.maxAge(30, TimeUnit.DAYS).cachePublic())
                .resourceChain(true)
                .addResolver(new VersionResourceResolver().addContentVersionStrategy("/**"));
    }
}
```

`VersionResourceResolver` 支持内容 hash 版本化（`style-abc123.css`），配合长缓存策略实现最优的静态资源缓存。

---

## 相关面试题 →

[[../../10_Developlanguage/005_Spring/02_SpringMVCSubject/09、文件上传与下载]]
[[../../10_Developlanguage/005_Spring/02_SpringMVCSubject/10、RESTful 风格]]
[[../../10_Developlanguage/005_Spring/02_SpringMVCSubject/11、静态资源处理]]
[[../../10_Developlanguage/005_Spring/02_SpringMVCSubject/12、跨域请求处理]]
