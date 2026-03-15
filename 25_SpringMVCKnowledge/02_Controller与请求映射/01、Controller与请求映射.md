# Controller 与请求映射

## @Controller 和 @RestController

`@Controller` 标识控制器组件，本质是 `@Component` 的特殊化形式，配合 `@ComponentScan` 被扫描进容器，方法返回的是视图名，经过 `ViewResolver` 解析后渲染页面。

`@RestController` = `@Controller` + `@ResponseBody`，方法返回值直接序列化写入 HTTP 响应体，不走视图解析，适合前后端分离的 RESTful API 开发。

简单说：传统服务端渲染用 `@Controller`，返回 JSON 用 `@RestController`。

---

## 请求映射注解

**`@RequestMapping`**：最基础的映射注解，支持类和方法两个级别。类级别定义 URL 前缀，方法级别细化路径。关键属性：
- `value/path`：URL 路径，支持 Ant 风格通配符（`?`单字符，`*`单级路径，`**`多级路径）
- `method`：HTTP 方法限定
- `consumes`：请求内容类型限定（`application/json`）
- `produces`：响应内容类型限定

HTTP 方法的快捷注解（推荐使用）：
- `@GetMapping`：查询操作，幂等，参数在 URL 中，有长度限制
- `@PostMapping`：创建操作，非幂等，参数在请求体中，无长度限制
- `@PutMapping`：完整更新，幂等，替换整个资源
- `@PatchMapping`：部分更新，只更新提供的字段
- `@DeleteMapping`：删除操作，幂等，成功返回 `204 No Content`

---

## 参数获取注解

**`@PathVariable`**：从 URL 路径模板中提取变量，RESTful API 必备：
```java
@GetMapping("/users/{id}")
public User getUser(@PathVariable Long id) { ... }

// 正则约束：只匹配数字
@GetMapping("/articles/{id:[0-9]+}")
public Article getArticle(@PathVariable Long id) { ... }
```

**`@RequestParam`**：从请求参数（查询字符串或表单参数）中提取值：
```java
// 基本用法
@GetMapping("/users")
public List<User> listUsers(@RequestParam String status,
                             @RequestParam(defaultValue = "1") int page,
                             @RequestParam(required = false) String keyword) { ... }

// 接收多值
@GetMapping("/filter")
public List<User> filter(@RequestParam List<Long> ids) { ... }
```

**`@RequestBody`**：将 HTTP 请求体（JSON/XML）反序列化为 Java 对象，通过 `HttpMessageConverter` 实现转换，常见是 `MappingJackson2HttpMessageConverter` 处理 JSON。

**`@ResponseBody`**：将方法返回值序列化写入响应体，根据请求的 `Accept` 头选择合适的 `HttpMessageConverter`。

**`@RequestHeader`**：从 HTTP 请求头提取值：
```java
@GetMapping("/info")
public String getInfo(@RequestHeader("Authorization") String auth,
                      @RequestHeader(value = "X-Request-ID", required = false) String traceId) { ... }
```

**`@CookieValue`**：从 HTTP Cookie 中提取值，通过 `CookieMethodArgumentResolver` 解析。

---

## @ModelAttribute

`@ModelAttribute` 在方法参数上用于表单数据绑定（将请求参数绑定到 Java 对象）；在方法上用于在每次请求前执行，将数据放入模型，在所有控制器方法前执行。

结合 `@SessionAttributes` 可以在多步骤表单（向导式）场景中跨请求保持数据，完成后调用 `SessionStatus.setComplete()` 清理会话。

---

## 相关面试题 →

[[../../10_Developlanguage/005_Spring/02_SpringMVCSubject/03、Controller 与请求映射]]
[[../../10_Developlanguage/005_Spring/02_SpringMVCSubject/10、RESTful 风格]]
