###### 1. 什么是 RESTful？

REST（Representational State Transfer，表述性状态转移）是 Roy Fielding 博士在 2000 年提出的软件架构风格，不是标准也不是协议，而是一套设计约束。遵循这套约束设计的 API 就叫 RESTful API。

它的核心思想是：把整个 Web 视为由**资源**构成的分布式系统，每个资源有唯一 URI 标识，客户端通过标准 HTTP 方法对资源进行 CRUD 操作，操作结果用 JSON 或 XML 等形式传递。

和 RPC 风格（`/getUser`、`/createOrder`）最大的区别是：RPC 关注**动作**（我要做什么），REST 关注**资源**（我要操作什么）。`GET /users/123` 比 `/getUser?id=123` 更直观，也更符合 HTTP 协议的语义。

📖 [[25_SpringMVCKnowledge/06_RESTful与文件处理/01、RESTful设计与文件处理]]

---

###### 2. RESTful 的设计原则有哪些？

REST 架构有六个核心约束：

**客户端-服务器分离**：前端展示和后端数据逻辑分离，各自独立演进。

**无状态通信**：每次请求必须包含完整的处理信息，服务器不保存客户端的会话状态。这是 REST 能横向扩展的基础——任意一台服务器都能处理任意请求。

**可缓存性**：响应要明确标注是否可缓存，允许缓存的资源减少网络请求，提升性能。

**统一接口**：这是 REST 最核心的约束，包含：资源有唯一 URI、用表述操作资源（JSON/XML）、消息自描述、以及 HATEOAS（超媒体作为应用状态引擎，响应中包含相关操作链接）。

**分层系统**：客户端不需要知道是在和真实服务器通信还是在和代理通信，系统可以分多层组织（负载均衡、缓存层等）。

**按需代码**（可选）：服务器可以给客户端发送可执行代码（如 JavaScript），扩展客户端功能。

📖 [[25_SpringMVCKnowledge/06_RESTful与文件处理/01、RESTful设计与文件处理]]

---

###### 3. SpringMVC 如何支持 RESTful？

Spring MVC 通过一套完整的注解体系和组件支持 RESTful 开发：

- `@RestController = @Controller + @ResponseBody`，所有方法返回值自动序列化为 JSON
- `@GetMapping` / `@PostMapping` / `@PutMapping` / `@PatchMapping` / `@DeleteMapping` 对应 HTTP 五种方法
- `@PathVariable` 提取 URL 路径变量（`/users/{id}`）
- `ResponseEntity` 精确控制 HTTP 状态码和响应头
- `HttpMessageConverter` 自动处理 JSON/XML 序列化反序列化

```java
@RestController
@RequestMapping("/api/users")
public class UserController {

    @GetMapping("/{userId}")
    public ResponseEntity<User> getUser(@PathVariable Long userId) {
        return userService.findById(userId)
            .map(ResponseEntity::ok)
            .orElse(ResponseEntity.notFound().build());
    }

    @PostMapping
    public ResponseEntity<User> createUser(@Valid @RequestBody User user) {
        User saved = userService.save(user);
        URI location = ServletUriComponentsBuilder.fromCurrentRequest()
            .path("/{id}").buildAndExpand(saved.getId()).toUri();
        return ResponseEntity.created(location).body(saved);
    }
}
```

📖 [[25_SpringMVCKnowledge/06_RESTful与文件处理/01、RESTful设计与文件处理]]

---

###### 4. @PutMapping 和 @DeleteMapping 的作用是什么？

两者都是 `@RequestMapping` 的组合注解，语义分别对应 HTTP PUT（完整更新）和 DELETE（删除）方法。

**PUT**：幂等操作，用于**完整替换**资源——客户端提供资源的完整表示，服务器用新内容覆盖旧内容。成功返回 200 或 204。

**DELETE**：幂等操作，删除指定资源。成功删除返回 204 No Content（不需要响应体）。

```java
@RestController
@RequestMapping("/api/users")
public class UserController {

    // PUT 完整更新：客户端提供所有字段
    @PutMapping("/{id}")
    public ResponseEntity<User> updateUser(
        @PathVariable Long id,
        @Valid @RequestBody User user) {
        user.setId(id);  // 确保 ID 一致性
        return ResponseEntity.ok(userService.update(user));
    }

    // DELETE 删除：成功返回 204，无响应体
    @DeleteMapping("/{id}")
    public ResponseEntity<Void> deleteUser(@PathVariable Long id) {
        userService.deleteById(id);
        return ResponseEntity.noContent().build();
    }
}
```

幂等性的意义：PUT 和 DELETE 执行多次结果相同，这让客户端在网络超时时可以安全重试，不用担心重复操作导致数据异常。

📖 [[25_SpringMVCKnowledge/06_RESTful与文件处理/01、RESTful设计与文件处理]]

---

###### 5. @PatchMapping 的作用是什么？

`@PatchMapping` 处理 HTTP PATCH 请求，用于**部分更新**资源，只传需要修改的字段，解决了 PUT 必须传完整资源的冗余问题。

比如只想改用户的手机号，PUT 要传所有字段，PATCH 只需传 `{"phone": "138xxx"}`，更节省带宽，也更清晰地表达了意图。

```java
@PatchMapping("/users/{id}")
public ResponseEntity<User> patchUser(
    @PathVariable Long id,
    @RequestBody Map<String, Object> updates) {

    User existing = userService.findById(id)
        .orElseThrow(() -> new ResourceNotFoundException("用户不存在"));

    // 只更新前端提交的字段
    updates.forEach((key, value) -> {
        switch (key) {
            case "name" -> existing.setName((String) value);
            case "email" -> existing.setEmail((String) value);
            case "phone" -> existing.setPhone((String) value);
            // 注意：不在白名单的字段直接忽略
        }
    });

    return ResponseEntity.ok(userService.update(existing));
}
```

PATCH 在 HTTP 规范中**不保证幂等性**（虽然实现上可以做到幂等），这点和 PUT 不同。

📖 [[25_SpringMVCKnowledge/06_RESTful与文件处理/01、RESTful设计与文件处理]]

---

###### 6. 如何处理 RESTful 风格的 URL？

RESTful URL 设计的核心原则：**用名词表示资源，用 HTTP 方法表示操作，用层级路径表示关系**。

```java
@RestController
@RequestMapping("/api")
public class ResourceController {

    // 单资源：/api/users/123
    @GetMapping("/users/{userId}")
    public User getUser(@PathVariable Long userId) {
        return userService.findById(userId);
    }

    // 嵌套资源（关联关系）：/api/users/123/orders/456
    @GetMapping("/users/{userId}/orders/{orderId}")
    public Order getOrder(
        @PathVariable Long userId,
        @PathVariable Long orderId) {
        return orderService.findUserOrder(userId, orderId);
    }

    // 正则约束路径变量格式：/api/products/book-123
    @GetMapping("/products/{categoryId:[a-z]+-[0-9]+}")
    public List<Product> getByCategory(@PathVariable String categoryId) {
        return productService.findByCategory(categoryId);
    }
}
```

实际项目中几个常见设计决策：

- 过滤/分页用查询参数：`GET /users?page=0&size=20&status=active`
- API 版本化：`/api/v1/users`（URL 版本化）或用请求头 `API-Version: v2`
- 资源集合操作：`POST /users/batch` 表示批量操作，不要用动词 URL

📖 [[25_SpringMVCKnowledge/06_RESTful与文件处理/01、RESTful设计与文件处理]]

---

###### 7. HiddenHttpMethodFilter 的作用是什么？

HTML 表单原生只支持 GET 和 POST 两种方法，无法直接发送 PUT、DELETE、PATCH 请求。`HiddenHttpMethodFilter` 提供了一个绕过方案：在 POST 表单里放一个隐藏字段 `_method`，过滤器把它转换为对应的 HTTP 方法。

```html
<!-- 前端表单：模拟 PUT 请求 -->
<form action="/users/123" method="post">
    <input type="hidden" name="_method" value="PUT">
    <input type="text" name="userName" value="新名字">
    <button type="submit">更新</button>
</form>
```

过滤器检测到 POST 请求且 `_method` 参数存在，就创建一个 Wrapper 覆盖 `getMethod()` 的返回值，让后续的 SpringMVC 处理器认为这是一个 PUT 请求：

```java
// Java Config 注册过滤器
@Bean
public FilterRegistrationBean<HiddenHttpMethodFilter> hiddenHttpMethodFilter() {
    FilterRegistrationBean<HiddenHttpMethodFilter> bean = new FilterRegistrationBean<>();
    bean.setFilter(new HiddenHttpMethodFilter());
    bean.addUrlPatterns("/*");
    return bean;
}
```

**实际上这个过滤器在现代前后端分离项目里基本用不到了**——Axios、Fetch API 原生支持发送任意 HTTP 方法，不需要这个绕过方案。它主要是为传统 JSP 表单提交场景设计的。

📖 [[25_SpringMVCKnowledge/06_RESTful与文件处理/01、RESTful设计与文件处理]]
