###### 1. @Controller 注解的作用是什么？

`@Controller` 是 Spring MVC 中标识控制器组件的核心注解，本质上是 `@Component` 的特殊化形式，标记的类会被组件扫描识别并注册为 Bean。

它的核心价值有三点：**组件扫描识别**、**请求映射处理**、**视图解析支持**。当 Spring 扫描到带有 `@Controller` 的类，会将其注册为 BeanDefinition；DispatcherServlet 初始化时，`RequestMappingHandlerMapping` 会扫描所有 `@Controller` 类中的 `@RequestMapping` 方法，建立 URL 到方法的映射关系；最后配合 `ViewResolver`，支持返回逻辑视图名并渲染具体模板。

```java
@Controller
@RequestMapping("/users")
public class UserController {
    @Autowired
    private UserService userService;

    @GetMapping("/{id}")
    public String getUser(@PathVariable Long id, Model model) {
        User user = userService.findById(id);
        model.addAttribute("user", user);
        return "user/detail";  // 解析为 /WEB-INF/views/user/detail.jsp
    }
}
```

📖 [[25_SpringMVCKnowledge/02_Controller与请求映射/01、Controller与请求映射]]

---

###### 2. @RequestMapping 注解的作用是什么？

`@RequestMapping` 是 Spring MVC 中定义请求映射关系的核心注解，用于将 HTTP 请求映射到具体的处理方法上，支持在类和方法两个层级使用。

它的关键属性包括：`value`（URL 路径）、`method`（HTTP 方法限制）、`params`（请求参数条件）、`headers`（请求头条件）、`consumes`（请求内容类型）、`produces`（响应内容类型）。

高级特性方面，支持 Ant 风格路径通配符（`?`、`*`、`**`）、路径变量 `{variable}`、以及一个方法同时映射多个路径。

```java
@Controller
@RequestMapping("/api")
public class ApiController {

    @RequestMapping(
        value = {"/users", "/members"},
        method = {RequestMethod.GET, RequestMethod.POST},
        consumes = MediaType.APPLICATION_JSON_VALUE,
        produces = MediaType.APPLICATION_JSON_VALUE
    )
    @ResponseBody
    public ResponseEntity<List<User>> getUsers() {
        // ...
    }
}
```

📖 [[25_SpringMVCKnowledge/02_Controller与请求映射/01、Controller与请求映射]]

---

###### 3. @GetMapping 和 @PostMapping 的区别是什么？

`@GetMapping` 和 `@PostMapping` 都是 `@RequestMapping` 的组合注解，分别固定了 GET 和 POST 方法，写起来更简洁，意图也更明确。

两者本质区别来自 HTTP 方法的语义差异：

**GET**：用于查询数据，参数拼接在 URL 中，受 URL 长度限制（约 2KB），是幂等操作，可以被浏览器缓存，参数对用户可见，相对不安全。

**POST**：用于提交、创建数据，参数放在请求体中，理论上无大小限制（受服务器配置约束），非幂等，不被缓存，参数不直接暴露，相对更安全。

```java
@RestController
@RequestMapping("/products")
public class ProductController {

    // GET：查询，参数走 URL
    @GetMapping("/{id}")
    public Product getProduct(@PathVariable Long id) {
        return productService.findById(id);
    }

    // POST：创建，参数走请求体
    @PostMapping
    public Product createProduct(@RequestBody Product product) {
        return productService.save(product);
    }
}
```

实际上 `@PutMapping`、`@PatchMapping`、`@DeleteMapping` 也是同样的组合注解模式，分别对应完整更新、部分更新和删除操作。

📖 [[25_SpringMVCKnowledge/02_Controller与请求映射/01、Controller与请求映射]]

---

###### 4. @PathVariable 注解的作用是什么？

`@PathVariable` 用于从 URL 路径模板中提取变量值并绑定到方法参数，是 RESTful API 开发的标配注解。

Spring 通过 `PathVariableMethodArgumentResolver` 解析它，从 URL 路径中提取对应的变量值后进行类型转换，传入方法参数。

```java
@RestController
public class PathVariableController {

    // 基础用法：从路径中取多个变量
    @GetMapping("/users/{userId}/orders/{orderId}")
    public Order getOrder(
        @PathVariable Long userId,
        @PathVariable("orderId") Long orderId) {
        return orderService.findUserOrder(userId, orderId);
    }

    // 高级用法：正则约束路径变量类型
    @GetMapping("/products/{category:[a-z]+}/{id:\\d+}")
    public Product getProductByCategory(
        @PathVariable String category,
        @PathVariable Integer id) {
        return productService.findByCategoryAndId(category, id);
    }

    // 用 Map 接收所有路径变量
    @GetMapping("/api/{version}/modules/{moduleName}/**")
    public Map<String, String> handlePathVariables(
        @PathVariable Map<String, String> pathVariables) {
        return pathVariables;
    }
}
```

📖 [[25_SpringMVCKnowledge/02_Controller与请求映射/01、Controller与请求映射]]

---

###### 5. @RequestParam 注解的作用是什么？

`@RequestParam` 用于从 URL 查询字符串或表单数据中提取参数值，绑定到方法参数上。

几个核心特性：通过 `required` 属性控制参数是否必传；通过 `defaultValue` 设置默认值；支持数组或 List 类型接收多个同名参数；也可以用 Map 接收所有请求参数。

```java
@RestController
public class RequestParamController {

    // 带默认值的分页参数
    @GetMapping("/users")
    public Page<User> getUsers(
        @RequestParam(defaultValue = "0") int page,
        @RequestParam(defaultValue = "20") int size,
        @RequestParam(defaultValue = "id") String sortBy) {
        return userService.findUsers(PageRequest.of(page, size, Sort.by(sortBy)));
    }

    // 多值参数：?ids=1,2,3 或 ?ids=1&ids=2
    @GetMapping("/products/batch")
    public List<Product> getProductsBatch(
        @RequestParam List<Long> ids) {
        return productService.findByIds(ids);
    }

    // Map 接收所有参数
    @PostMapping("/filter")
    public List<Product> filterProducts(
        @RequestParam Map<String, String> filters) {
        return productService.filter(filters);
    }
}
```

📖 [[25_SpringMVCKnowledge/02_Controller与请求映射/01、Controller与请求映射]]

---

###### 6. @RequestBody 注解的作用是什么？

`@RequestBody` 用于将 HTTP 请求体的内容绑定到方法参数，主要处理 JSON、XML 等非表单格式的数据。

它的工作原理是：根据请求的 `Content-Type` 选择合适的 `HttpMessageConverter`，将请求体反序列化为目标 Java 对象。比如 JSON 数据用 `MappingJackson2HttpMessageConverter` 处理，XML 用 `Jaxb2RootElementHttpMessageConverter`。

需要注意，使用 `@RequestBody` 时，请求头必须设置 `Content-Type: application/json`，否则转换器无法匹配。

```java
@RestController
public class RequestBodyController {

    // 接收 JSON 对象，并结合 @Valid 做数据校验
    @PostMapping("/users")
    public User createUser(@RequestBody @Valid User user) {
        return userService.save(user);
    }

    // 接收 JSON 数组
    @PostMapping("/orders/batch")
    public ResponseEntity<String> createOrders(
        @RequestBody @Valid List<Order> orders) {
        orderService.batchCreate(orders);
        return ResponseEntity.ok("批量创建成功");
    }
}
```

📖 [[25_SpringMVCKnowledge/02_Controller与请求映射/01、Controller与请求映射]]

---

###### 7. @ResponseBody 注解的作用是什么？

`@ResponseBody` 用于将方法的返回值直接写入 HTTP 响应体，而不是被解析为视图名称，是构建 RESTful API 的关键注解。

处理流程是：`RequestMappingHandlerAdapter` 检测到 `@ResponseBody` 后，根据请求的 `Accept` 头选择合适的 `HttpMessageConverter`，将返回值序列化写入响应流。

`@RestController` 就是 `@Controller + @ResponseBody` 的组合，类上加了 `@RestController` 就等于所有方法都默认带了 `@ResponseBody`。

```java
@Controller
public class ResponseBodyController {

    // 返回字符串直接写入响应体
    @ResponseBody
    @GetMapping("/status")
    public String getStatus() {
        return "服务运行正常";
    }

    // 返回对象自动序列化为 JSON
    @ResponseBody
    @GetMapping("/user/{id}")
    public User getUser(@PathVariable Long id) {
        return userService.findById(id);
    }

    // 带 Location 响应头的创建接口
    @ResponseBody
    @PostMapping("/users")
    public ResponseEntity<User> createUser(@RequestBody User user) {
        User savedUser = userService.save(user);
        URI location = ServletUriComponentsBuilder
            .fromCurrentRequest()
            .path("/{id}")
            .buildAndExpand(savedUser.getId())
            .toUri();
        return ResponseEntity.created(location).body(savedUser);
    }
}
```

📖 [[25_SpringMVCKnowledge/02_Controller与请求映射/01、Controller与请求映射]]

---

###### 8. @RestController 和 @Controller 的区别是什么？

这两个注解最核心的区别就一点：**返回值的处理方式不同**。

`@Controller` 方法的返回值默认被当作视图名称，由 `ViewResolver` 解析为具体的模板页面，适合传统的服务端渲染（SSR）场景。如果某个方法需要直接返回数据，要单独加 `@ResponseBody`。

`@RestController` 是 `@Controller + @ResponseBody` 的组合注解，所有方法默认都有 `@ResponseBody` 语义，返回值直接通过 `HttpMessageConverter` 序列化写入响应体，适合前后端分离的 RESTful API 场景。

```java
// 传统控制器，返回视图名
@Controller
@RequestMapping("/web")
public class WebController {

    @GetMapping("/users")
    public String userList(Model model) {
        model.addAttribute("users", userService.findAll());
        return "user/list";  // 解析为模板页面
    }
}

// REST 控制器，直接返回 JSON
@RestController
@RequestMapping("/api")
public class ApiController {

    @GetMapping("/users")
    public List<User> getUsers() {
        return userService.findAll();
    }
}
```

一个项目里两种控制器完全可以并存，页面渲染的走 `@Controller`，接口数据的走 `@RestController`，各司其职。

📖 [[25_SpringMVCKnowledge/02_Controller与请求映射/01、Controller与请求映射]]

---

###### 9. 如何在 SpringMVC 中处理 RESTful 请求？

RESTful API 的核心思想是**资源导向**和**HTTP 方法语义化**：用名词表示资源，用 HTTP 方法表达操作意图，用状态码传递结果。

几个关键原则：URL 只用名词标识资源（如 `/api/users/{id}`），GET 查询、POST 创建、PUT 完整更新、PATCH 部分更新、DELETE 删除，每次请求携带完整信息，服务器不保存会话状态。

```java
@RestController
@RequestMapping("/api/users")
public class UserRestController {

    // GET 查询列表，支持分页
    @GetMapping
    public ResponseEntity<List<User>> getAllUsers(
        @RequestParam(defaultValue = "0") int page,
        @RequestParam(defaultValue = "20") int size) {
        Page<User> users = userService.findAll(PageRequest.of(page, size));
        return ResponseEntity.ok()
            .header("X-Total-Count", String.valueOf(users.getTotalElements()))
            .body(users.getContent());
    }

    // GET 查询单个
    @GetMapping("/{id}")
    public ResponseEntity<User> getUser(@PathVariable Long id) {
        return userService.findById(id)
            .map(ResponseEntity::ok)
            .orElse(ResponseEntity.notFound().build());
    }

    // POST 创建，返回 201 + Location 头
    @PostMapping
    public ResponseEntity<User> createUser(@Valid @RequestBody User user) {
        User savedUser = userService.save(user);
        URI location = ServletUriComponentsBuilder.fromCurrentRequest()
            .path("/{id}").buildAndExpand(savedUser.getId()).toUri();
        return ResponseEntity.created(location).body(savedUser);
    }

    // PUT 完整更新
    @PutMapping("/{id}")
    public ResponseEntity<User> updateUser(
        @PathVariable Long id,
        @Valid @RequestBody User user) {
        user.setId(id);
        return ResponseEntity.ok(userService.update(user));
    }

    // PATCH 部分更新
    @PatchMapping("/{id}")
    public ResponseEntity<User> patchUser(
        @PathVariable Long id,
        @RequestBody Map<String, Object> updates) {
        return ResponseEntity.ok(userService.patch(id, updates));
    }

    // DELETE 删除，返回 204
    @DeleteMapping("/{id}")
    public ResponseEntity<Void> deleteUser(@PathVariable Long id) {
        userService.delete(id);
        return ResponseEntity.noContent().build();
    }
}
```

📖 [[25_SpringMVCKnowledge/06_RESTful与文件处理/01、RESTful设计与文件处理]]

---

###### 10. @RequestHeader 注解的作用是什么？

`@RequestHeader` 用于从 HTTP 请求头中提取指定值并绑定到方法参数，常见场景包括读取 `User-Agent`、`Authorization`、`Accept-Language`、自定义追踪 ID 等。

```java
@RestController
public class HeaderController {

    // 读取指定请求头，带默认值
    @GetMapping("/info")
    public String getHeaderInfo(
        @RequestHeader("User-Agent") String userAgent,
        @RequestHeader(value = "Accept-Language", defaultValue = "zh-CN") String language) {
        return String.format("客户端: %s, 语言: %s", userAgent, language);
    }

    // 基于请求头做 API 版本控制
    @GetMapping("/versioned")
    public ResponseEntity<String> versionedEndpoint(
        @RequestHeader(value = "API-Version", defaultValue = "v1") String apiVersion) {
        return switch (apiVersion) {
            case "v1" -> ResponseEntity.ok("API 版本 1 响应");
            case "v2" -> ResponseEntity.ok("API 版本 2 响应");
            default -> ResponseEntity.badRequest().body("不支持的 API 版本");
        };
    }
}
```

📖 [[25_SpringMVCKnowledge/02_Controller与请求映射/01、Controller与请求映射]]

---

###### 11. @CookieValue 注解的作用是什么？

`@CookieValue` 用于从 HTTP Cookie 中提取指定值并绑定到方法参数，底层通过 `CookieMethodArgumentResolver` 从 `HttpServletRequest.getCookies()` 中查找匹配项后进行类型转换。

```java
@RestController
public class CookieController {

    // 提取 Cookie 值，带默认值
    @GetMapping("/user")
    public String getUserInfo(
        @CookieValue("sessionId") String sessionId,
        @CookieValue(value = "theme", defaultValue = "light") String theme) {
        User user = sessionService.getUserBySession(sessionId);
        return String.format("用户: %s, 主题: %s", user.getName(), theme);
    }

    // 登录时设置 Cookie
    @PostMapping("/login")
    public ResponseEntity<String> login(@RequestBody LoginRequest request,
                                        HttpServletResponse response) {
        User user = authService.authenticate(request);
        String sessionId = sessionService.createSession(user);

        Cookie cookie = new Cookie("sessionId", sessionId);
        cookie.setHttpOnly(true);
        cookie.setSecure(true);
        cookie.setMaxAge(7 * 24 * 60 * 60);  // 7 天
        cookie.setPath("/");
        response.addCookie(cookie);

        return ResponseEntity.ok("登录成功");
    }

    // 退出时删除 Cookie
    @PostMapping("/logout")
    public ResponseEntity<String> logout(
        @CookieValue("sessionId") String sessionId,
        HttpServletResponse response) {
        sessionService.invalidateSession(sessionId);

        Cookie cookie = new Cookie("sessionId", "");
        cookie.setMaxAge(0);  // 立即过期
        cookie.setPath("/");
        response.addCookie(cookie);

        return ResponseEntity.ok("退出成功");
    }
}
```

📖 [[25_SpringMVCKnowledge/07_配置扩展与整合/01、配置扩展与技术整合]]

---

###### 12. @ModelAttribute 注解的作用是什么？

`@ModelAttribute` 用于将请求参数绑定到模型对象，在传统的表单处理场景中特别有用，支持参数绑定和数据预加载两种用法。

**参数级用法**：将表单数据绑定到 POJO 对象并自动加入 Model。
**方法级用法**：在每个请求处理方法执行前先执行，用来预加载公共数据（比如下拉列表的选项数据）。

```java
@Controller
@RequestMapping("/products")
public class ProductController {

    // 方法级：每次请求前先加载分类列表
    @ModelAttribute("categories")
    public List<Category> loadCategories() {
        return categoryService.findAll();
    }

    // 参数级：表单提交，绑定表单数据并校验
    @PostMapping
    public String createProduct(
        @ModelAttribute("product") @Valid Product product,
        BindingResult result) {
        if (result.hasErrors()) {
            return "products/form";
        }
        productService.save(product);
        return "redirect:/products";
    }

    // 配合 @InitBinder 禁止绑定 ID 字段，防止用户篡改
    @InitBinder
    public void initBinder(WebDataBinder binder) {
        binder.setDisallowedFields("id");
    }
}
```

📖 [[25_SpringMVCKnowledge/02_Controller与请求映射/01、Controller与请求映射]]

---

###### 13. @SessionAttributes 注解的作用是什么？

`@SessionAttributes` 用于在 Session 范围内保存指定的模型属性，实现跨请求的数据持久化，适合多步骤表单这类场景，不用每一步都重新从数据库查数据。

标注了 `@SessionAttributes("order")` 后，Spring 会自动把 Model 中名为 `order` 的对象存入 HttpSession，后续请求也能从 Session 中恢复这个对象。用 `SessionStatus.setComplete()` 可以通知 Spring 清理这些会话属性。

```java
@Controller
@RequestMapping("/order")
@SessionAttributes("order")  // order 对象在 Session 中保持
public class OrderController {

    // 第一步：初始化订单
    @GetMapping("/create")
    public String createOrderForm(Model model) {
        if (!model.containsAttribute("order")) {
            model.addAttribute("order", new Order());
        }
        return "order/step1";
    }

    @PostMapping("/step1")
    public String processStep1(
        @ModelAttribute("order") @Valid Order order,
        BindingResult result) {
        if (result.hasErrors()) return "order/step1";
        return "redirect:/order/step2";
    }

    // 最后一步：提交并清理 Session
    @PostMapping("/confirm")
    public String confirmOrder(
        @ModelAttribute("order") Order order,
        SessionStatus status) {
        Order savedOrder = orderService.save(order);
        status.setComplete();  // 清理 Session 中的 order 属性
        return "redirect:/order/complete/" + savedOrder.getId();
    }
}
```

📖 [[25_SpringMVCKnowledge/07_配置扩展与整合/01、配置扩展与技术整合]]
