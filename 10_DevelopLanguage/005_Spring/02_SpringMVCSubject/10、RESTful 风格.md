###### 1. 什么是 RESTful?
RESTful是一种**软件架构风格**而非标准或协议，其核心思想源于Roy Fielding博士在2000年提出的表述性状态转移（Representational State Transfer）架构原则。这种风格为网络应用程序提供了**统一接口约束**，使系统组件能够通过标准的HTTP方法对资源进行操作。
从本质上看，RESTful将整个互联网视为由**资源**构成的分布式系统，每个资源通过URI唯一标识。客户端通过HTTP协议定义的标准方法（GET、POST、PUT、DELETE等）操作这些资源，操作结果通过资源的不同**表述形式**（如JSON、XML）在客户端和服务器之间传递。
与传统RPC风格相比，RESTful的核心区别在于**关注点**的不同：RPC风格关注方法调用（如/getUser、/createOrder），而RESTful风格关注资源状态管理（如对/user/123使用GET获取、PUT更新）。这种资源导向的设计使API更加简洁、可预测且易于缓存，特别适合构建可扩展的Web服务。
###### 2. RESTful 的设计原则有哪些?
RESTful架构遵循六个核心约束条件，这些约束共同构成了REST风格的基础：
1. **客户端-服务器分离**：前端展示逻辑与后端数据存储逻辑分离，允许两者独立进化
2. **无状态通信**：每个请求必须包含处理所需的所有信息，服务器不保存客户端会话状态。这使得系统更适合横向扩展和云计算环境
3. **可缓存性**：响应必须明确标示自身是否可缓存，以减少网络往返，提高性能
4. **统一接口**：这是REST最核心的约束，包含四个子原则：
    - **资源标识**：每个资源有唯一URI（如`/users/123`）
    - **通过表述操作资源**：客户端通过资源表述（如JSON、XML）来操作资源状态
    - **自描述消息**：每个消息包含足够信息描述如何处理该消息
    - **超媒体作为应用状态引擎**：响应中包含相关资源的链接，引导客户端发现可用操作
5. **分层系统**：系统可以由多个层次组成，每个层次只需了解相邻层次，提高系统组件的独立性和可维护性
6. **按需代码**：服务器可以临时扩展客户端功能（如通过JavaScript），这是唯一可选约束
###### 3. SpringMVC 如何支持 RESTful?
Spring MVC通过一系列注解和组件为RESTful开发提供**全面支持**，其核心设计理念是将HTTP协议特性与Java注解完美结合。
**核心组件架构：**
1. **注解驱动模型**：Spring MVC提供`@RestController`、`@RequestMapping`及其变体等注解，将普通Java方法转化为REST端点
2. **内容协商机制**：通过`ContentNegotiationManager`和`HttpMessageConverter`体系，自动处理多种数据格式（JSON/XML）的请求与响应
3. **URL路由解析**：`@PathVariable`注解支持将URL模板变量绑定到方法参数
4. **响应状态管理**：`ResponseEntity`和`@ResponseStatus`提供对HTTP状态码的精细控制
**源码级实现示例：**
```java
@RestController  // 组合注解：@Controller + @ResponseBody
@RequestMapping("/api/users")
public class UserController {
    
    @Autowired
    private UserService userService;
    
    // HTTP方法到Java方法的直接映射
    @GetMapping("/{userId}")
    public ResponseEntity<User> getUser(@PathVariable Long userId) {
        User user = userService.findById(userId);
        if(user != null) {
            return ResponseEntity.ok(user);  // 200 OK + 用户数据
        }
        return ResponseEntity.notFound().build();  // 404 Not Found
    }
}
```
在`DispatcherServlet`的请求处理流程中，`RequestMappingHandlerMapping`负责将HTTP请求映射到具体的处理器方法，而`RequestMappingHandlerAdapter`则负责调用目标方法并处理参数绑定。
###### 4. @PutMapping 和 @DeleteMapping 的作用是什么?
`@PutMapping`和`@DeleteMapping`是Spring MVC提供的**组合注解**，分别用于处理HTTP PUT和DELETE请求，遵循RESTful架构中对资源更新和删除操作的语义定义。
**@PutMapping - 完整资源更新：**
- **设计语义**：PUT方法具有**幂等性**，多次相同请求产生的结果一致。它用于**替换**整个资源，要求客户端提供资源的完整表示
- **源码实现**：`@PutMapping`本质上是`@RequestMapping(method = RequestMethod.PUT)`的简写形式
```java
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
@RequestMapping(method = RequestMethod.PUT)  // 元注解定义
public @interface PutMapping {
    // 属性继承自@RequestMapping
}
```
- **使用场景**：
```java
@PutMapping("/users/{id}")
public ResponseEntity<User> updateUser(@PathVariable Long id, 
                                     @RequestBody User user) {
    // 实现完整更新逻辑
    user.setId(id);  // 确保ID一致性
    User updatedUser = userService.update(user);
    return ResponseEntity.ok(updatedUser);
}
```
**@DeleteMapping - 资源删除：**
- **设计语义**：DELETE方法同样具有**幂等性**，用于删除指定资源。成功执行后通常返回`204 No Content`状态
- **实战应用**：
```java
@DeleteMapping("/users/{id}")
public ResponseEntity<Void> deleteUser(@PathVariable Long id) {
    userService.deleteById(id);
    return ResponseEntity.noContent().build();  // 204响应，无内容体
}
```
**幂等性保障**：由于PUT和DELETE的幂等特性，Spring MVC无需额外处理重复请求问题，这与非幂等的POST方法形成鲜明对比。
###### 5. @PatchMapping 的作用是什么?
`@PatchMapping`用于处理HTTP PATCH请求，实现资源的**部分更新**而非完整替换，解决了PUT方法在只需修改少量字段时的过度传输问题。
**与PUT的关键差异：**
- **更新粒度**：PUT替换整个资源，PATCH仅更新提供的字段
- **数据传输**：PATCH只需传递需要修改的字段，减少网络开销
- **业务处理**：PATCH需要更复杂的业务逻辑来处理部分更新
**源码级实现策略：**
```java
@PatchMapping("/users/{id}")
public ResponseEntity<User> patchUser(@PathVariable Long id, 
                                    @RequestBody Map<String, Object> updates) {
    User existingUser = userService.findById(id);
    
    // 反射或业务逻辑实现部分字段更新
    updates.forEach((key, value) -> {
        switch(key) {
            case "name": existingUser.setName((String)value); break;
            case "email": existingUser.setEmail((String)value); break;
            // 其他字段处理...
        }
    });
    
    User updatedUser = userService.update(existingUser);
    return ResponseEntity.ok(updatedUser);
}
```
**高级实现 - DTO模式：**
```java
@PatchMapping("/users/{id}")
public ResponseEntity<User> patchUser(@PathVariable Long id, 
                                    @RequestBody UserPatchDTO patchDTO) {
    // 使用自定义DTO精确控制可更新字段
    User updatedUser = userService.partialUpdate(id, patchDTO);
    return ResponseEntity.ok(updatedUser);
}
```
**设计价值**：`@PatchMapping`提供了更精细的资源操作粒度，符合API设计的最佳实践，特别适合移动端等网络环境受限的场景。
###### 6. 如何处理 RESTful 风格的 URL?
RESTful URL的核心特征是**资源导向**和**层级结构**，Spring MVC通过多种机制提供支持。
**URL模板变量与@PathVariable：**
```java
@RestController
@RequestMapping("/api")
public class ResourceController {
    
    // 单级路径参数：/api/users/123
    @GetMapping("/users/{userId}")
    public User getUser(@PathVariable("userId") Long id) {
        return userService.findById(id);
    }
    
    // 多级路径参数：/api/users/123/orders/456
    @GetMapping("/users/{userId}/orders/{orderId}")
    public Order getOrder(@PathVariable Long userId, 
                         @PathVariable Long orderId) {
        return orderService.findUserOrder(userId, orderId);
    }
    
    // 正则约束路径参数：/api/products/category-123
    @GetMapping("/products/{categoryId:[a-z]+-[0-9]+}")
    public List<Product> getByCategory(@PathVariable String categoryId) {
        return productService.findByCategory(categoryId);
    }
}
```
**源码解析**：Spring MVC通过`PathVariableMethodArgumentResolver`解析URL模板变量。在`HandlerMethod`执行时，该解析器会从`UrlPathHelper`中提取路径变量值并完成类型转换。
**矩阵变量与复杂URL参数：**
```java
// 处理矩阵变量：/api/products;category=books;price<100
@GetMapping("/products/{productId}")
public Product getProduct(@MatrixVariable Map<String, String> matrixVars,
                        @PathVariable String productId) {
    // matrixVars包含category=books, price<100等参数
    return productService.findWithFilters(productId, matrixVars);
}
```
**设计原则**：RESTful URL应使用名词而非动词，保持层级结构清晰，版本化API（如`/api/v1/users`），并通过查询参数处理过滤、分页等复杂场景。
###### 7. HiddenHttpMethodFilter 的作用是什么?
`HiddenHttpMethodFilter`是Spring MVC提供的**解决方案**，用于处理浏览器对PUT、DELETE、PATCH等HTTP方法的**不支持问题**。
**问题背景**：HTML表单原生只支持GET和POST方法，为在Web浏览器中实现完整的RESTful操作，需要一种转换机制。
**工作原理**：该过滤器通过检查POST请求中的隐藏参数`_method`，将其**转换为目标HTTP方法**：
```xml
<!-- 前端表单示例 -->
<form action="/users/123" method="post">
    <input type="hidden" name="_method" value="PUT">
    <input type="text" name="userName" value="新姓名">
    <button type="submit">更新用户</button>
</form>
```
**源码分析**：`HiddenHttpMethodFilter`继承自`OncePerRequestFilter`，在`doFilterInternal`方法中实现方法转换：
```java
public class HiddenHttpMethodFilter extends OncePerRequestFilter {
    protected void doFilterInternal(HttpServletRequest request, 
                                  HttpServletResponse response, 
                                  FilterChain filterChain) {
        // 1. 检查是否为POST请求且_method参数存在
        if ("POST".equals(request.getMethod()) && request.getParameter("_method") != null) {
            // 2. 获取_method参数值（PUT/DELETE/PATCH等）
            String method = request.getParameter("_method").toUpperCase(Locale.ENGLISH);
            // 3. 创建HttpMethodRequestWrapper包装原始请求
            HttpMethodRequestWrapper wrapper = new HttpMethodRequestWrapper(request, method);
            // 4. 将包装后请求传递到后续过滤器链
            filterChain.doFilter(wrapper, response);
        } else {
            filterChain.doFilter(request, response);
        }
    }
}
```
**配置方式**：
```xml
<!-- web.xml配置 -->
<filter>
    <filter-name>HiddenHttpMethodFilter</filter-name>
    <filter-class>org.springframework.web.filter.HiddenHttpMethodFilter</filter-class>
</filter>
<filter-mapping>
    <filter-name>HiddenHttpMethodFilter</filter-name>
    <url-pattern>/*</url-pattern>
</filter-mapping>
```
```java
// Java Config配置
@Configuration
public class WebConfig {
    @Bean
    public FilterRegistrationBean<HiddenHttpMethodFilter> hiddenHttpMethodFilter() {
        FilterRegistrationBean<HiddenHttpMethodFilter> bean = new FilterRegistrationBean<>();
        bean.setFilter(new HiddenHttpMethodFilter());
        bean.addUrlPatterns("/*");
        return bean;
    }
}
```
**限制与注意事项**：此方案主要适用于传统Web应用（表单提交），现代前后端分离架构通常直接使用fetch API或Axios等库发送PUT/DELETE请求，无需此过滤器。