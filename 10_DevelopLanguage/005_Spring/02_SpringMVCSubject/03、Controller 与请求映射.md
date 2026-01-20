###### 1. @Controller 注解的作用是什么?
@Controller是Spring MVC框架中**标识控制器组件**的核心注解，它标记的类负责处理HTTP请求并返回响应结果。从源码角度看，@Controller是一个**元注解**，其本质是@Component注解的特殊化形式。
**源码级分析：**
```java
@Target({ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Component  // 关键：继承自@Component，参与组件扫描
public @interface Controller {
    String value() default "";  // 可指定Bean名称
}
```
**核心作用机制：**
1. **组件扫描识别**：Spring通过ClassPathBeanDefinitionScanner扫描时，识别带有@Controller注解的类并注册为BeanDefinition
2. **请求映射处理**：DispatcherServlet初始化时，RequestMappingHandlerMapping会扫描所有@Controller类中的@RequestMapping方法，建立URL到方法的映射关系
3. **视图解析支持**：与ViewResolver配合，支持返回逻辑视图名并渲染具体视图模板
**实战示例：**
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
        return "user/detail";  // 解析为/WEB-INF/views/user/detail.jsp
    }
}
```
###### 2. @RequestMapping 注解的作用是什么?
@RequestMapping是Spring MVC中**定义请求映射关系**的核心注解，用于将HTTP请求映射到特定的处理器方法。它支持在类和方法级别使用，提供精细的URL匹配规则。
**源码中的关键属性：**
```java
public @interface RequestMapping {
    String[] value() default {};                 // URL路径映射
    RequestMethod[] method() default {};         // HTTP方法限制
    String[] params() default {};                // 请求参数条件
    String[] headers() default {};                // 请求头条件
    String[] consumes() default {};              // 内容类型限制
    String[] produces() default {};              // 响应类型限制
}
```
**高级映射特性：**
- **Ant风格路径**：支持?, *, **通配符匹配
- **路径变量**：通过{variable}定义RESTful路径参数
- **多重映射**：一个方法可同时映射多个URL路径
**配置示例：**
```java
@Controller
@RequestMapping("/api")  // 类级别基础路径
public class ApiController {
    
    @RequestMapping(
        value = {"/users", "/members"},  // 多路径映射
        method = {RequestMethod.GET, RequestMethod.POST},
        consumes = MediaType.APPLICATION_JSON_VALUE,
        produces = MediaType.APPLICATION_JSON_VALUE
    )
    @ResponseBody
    public ResponseEntity<List<User>> getUsers() {
        // 处理方法
    }
}
```
###### 3. @GetMapping 和 @PostMapping 的区别是什么?
@GetMapping和@PostMapping是@RequestMapping的**组合注解**，分别专门处理GET和POST请求，使代码更加简洁意图明确。
**源码实现对比：**
```java
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@RequestMapping(method = RequestMethod.GET)  // 固定为GET方法
public @interface GetMapping { /* 属性同@RequestMapping */ }

@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
@Documented  
@RequestMapping(method = RequestMethod.POST)  // 固定为POST方法
public @interface PostMapping { /* 属性同@RequestMapping */ }
```
**核心区别分析：**

|**特性**​|**@GetMapping**​|**@PostMapping**​|
|---|---|---|
|**HTTP方法**​|GET|POST|
|**参数传递**​|URL查询字符串或路径变量|请求体(Form-data/JSON)|
|**安全性**​|参数暴露在URL中，相对不安全|参数在请求体中，相对安全|
|**数据长度**​|受URL长度限制(约2KB)|理论无限制(服务器配置限制)|
|**幂等性**​|幂等操作(可重复执行)|非幂等操作(可能改变状态)|
|**缓存**​|可被缓存|通常不被缓存|
|**使用场景**​|数据查询、页面跳转|数据提交、资源创建、敏感操作|
**实战应用示例：**
```java
@RestController
@RequestMapping("/products")
public class ProductController {
    
    // GET请求：查询产品信息
    @GetMapping("/{id}")
    public Product getProduct(@PathVariable Long id) {
        return productService.findById(id);
    }
    
    // POST请求：创建新产品
    @PostMapping
    public Product createProduct(@RequestBody Product product) {
        return productService.save(product);
    }
}
```
###### 4. @PathVariable 注解的作用是什么?
@PathVariable用于**从URL路径模板中提取变量值**并绑定到方法参数，是RESTful API开发的核心注解。
**源码机制：**
Spring通过`PathVariableMethodArgumentResolver`解析@PathVariable注解。在`HandlerMethod`执行时，参数解析器会从`UrlPathHelper`中提取路径变量值并进行类型转换。
**高级用法：**
```java
@RestController
public class PathVariableController {
    
    // 基础路径变量
    @GetMapping("/users/{userId}/orders/{orderId}")
    public Order getOrder(
        @PathVariable Long userId,
        @PathVariable("orderId") Long orderId) {  // 显式指定变量名
        return orderService.findUserOrder(userId, orderId);
    }
    
    // 正则表达式约束路径变量
    @GetMapping("/products/{category:[a-z]+}/{id:\\d+}")
    public Product getProductByCategory(
        @PathVariable String category,
        @PathVariable Integer id) {
        return productService.findByCategoryAndId(category, id);
    }
    
    // Map接收所有路径变量
    @GetMapping("/api/{version}/modules/{moduleName}/**")
    public Map<String, String> handlePathVariables(
        @PathVariable Map<String, String> pathVariables) {
        return pathVariables;  // 返回所有路径变量键值对
    }
}
```
###### 5. @RequestParam 注解的作用是什么?
@RequestParam用于**从请求参数中提取值**并绑定到方法参数，支持查询字符串和表单数据。
**核心特性：**
- **参数来源**：从URL查询字符串(?name=value)或表单数据中获取值
- **默认值支持**：通过defaultValue属性设置参数默认值
- **可选性控制**：通过required属性控制参数是否必须
- **多值支持**：支持数组或List类型接收多个同名参数值
**源码级参数解析：**
Spring通过`RequestParamMethodArgumentResolver`处理@RequestParam注解。解析器从`ServletRequest.getParameter()`或`getParameterValues()`方法中获取参数值，并使用`DataBinder`进行类型转换。
**实战示例：**
```java
@RestController
public class RequestParamController {
    
    // 基础用法 - 必需参数
    @GetMapping("/search")
    public List<Product> searchProducts(
        @RequestParam String keyword,
        @RequestParam(required = false) String category) {
        return productService.search(keyword, category);
    }
    
    // 带默认值的可选参数
    @GetMapping("/users")
    public Page<User> getUsers(
        @RequestParam(defaultValue = "0") int page,
        @RequestParam(defaultValue = "20") int size,
        @RequestParam(defaultValue = "id") String sortBy) {
        return userService.findUsers(PageRequest.of(page, size, Sort.by(sortBy)));
    }
    
    // 多值参数（如?ids=1,2,3或?ids=1&ids=2）
    @GetMapping("/products/batch")
    public List<Product> getProductsBatch(
        @RequestParam List<Long> ids) {  // 自动转换为List
        return productService.findByIds(ids);
    }
    
    // Map接收所有请求参数
    @PostMapping("/filter")
    public List<Product> filterProducts(
        @RequestParam Map<String, String> filters) {
        return productService.filter(filters);
    }
}
```
###### 6. @RequestBody 注解的作用是什么?
@RequestBody用于**将HTTP请求体内容绑定到方法参数**，主要用于处理JSON/XML等非表单数据。
**工作原理：**
1. **内容协商**：根据Content-Type头选择适当的HttpMessageConverter
2. **数据绑定**：使用选择的转换器(如MappingJackson2HttpMessageConverter)将请求体转换为目标类型
3. **数据验证**：配合@Valid注解实现自动数据验证
**支持的HttpMessageConverter：**
- **MappingJackson2HttpMessageConverter**：处理application/json
- **Jaxb2RootElementHttpMessageConverter**：处理application/xml
- **StringHttpMessageConverter**：处理text/plain
- **FormHttpMessageConverter**：处理multipart/form-data
**高级应用：**
```java
@RestController
public class RequestBodyController {
    
    // 基础JSON绑定
    @PostMapping("/users")
    public User createUser(@RequestBody @Valid User user) {
        return userService.save(user);
    }
    
    // 自定义类型解析
    @PostMapping("/orders/batch")
    public ResponseEntity<String> createOrders(
        @RequestBody @Valid List<Order> orders) {
        orderService.batchCreate(orders);
        return ResponseEntity.ok("批量创建成功");
    }
    
    // 获取原始请求体
    @PostMapping("/raw")
    public void handleRawBody(@RequestBody String rawBody) {
        // 处理原始字符串数据
        log.info("原始请求体: {}", rawBody);
    }
    
    // 配合HttpEntity使用
    @PostMapping("/entity")
    public ResponseEntity<User> updateUser(HttpEntity<User> httpEntity) {
        User user = httpEntity.getBody();
        HttpHeaders headers = httpEntity.getHeaders();
        // 可同时访问请求头和请求体
        return ResponseEntity.ok(userService.update(user));
    }
}
```
###### 7. @ResponseBody 注解的作用是什么?
@ResponseBody用于**将方法返回值直接写入HTTP响应体**，而非视图名称。它是构建RESTful API的关键注解。
**响应处理流程：**
1. **返回值处理**：RequestMappingHandlerAdapter检测方法是否有@ResponseBody注解
2. **消息转换**：根据Accept头选择适当的HttpMessageConverter
3. **内容写入**：将返回值通过转换器写入HttpServletResponse输出流
**与@RestController的关系：**
@RestController是一个组合注解，相当于@Controller + @ResponseBody，表示该类所有方法都默认使用@ResponseBody语义。
**深度应用示例：**
```java
@Controller
public class ResponseBodyController {
    
    // 基本数据类型响应
    @ResponseBody
    @GetMapping("/status")
    public String getStatus() {
        return "服务运行正常";
    }
    
    // 对象自动序列化为JSON
    @ResponseBody
    @GetMapping("/user/{id}")
    public User getUser(@PathVariable Long id) {
        return userService.findById(id);
    }
    
    // 自定义HTTP状态码和头信息
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
    
    // 异步响应处理
    @ResponseBody
    @GetMapping("/async")
    public Callable<String> asyncProcessing() {
        return () -> {
            Thread.sleep(5000);  // 模拟耗时操作
            return "异步处理完成";
        };
    }
}
```
###### 8. @RestController 和 @Controller 的区别是什么?
@RestController和@Controller是Spring MVC中**两种不同的控制器注解**，主要区别在于响应处理方式和适用场景。
**核心区别对比表：**

|**特性**​|**@Controller**​|**@RestController**​|
|---|---|---|
|**注解组合**​|单独的@Controller注解|@Controller + @ResponseBody组合|
|**返回值处理**​|返回视图名称，由ViewResolver解析|返回值直接写入响应体，不进行视图解析|
|**默认响应类型**​|HTML视图渲染|JSON/XML等数据格式|
|**适用场景**​|传统服务端渲染应用|RESTful API、前后端分离应用|
|**方法级控制**​|需要配合@ResponseBody返回数据|所有方法自动具有@ResponseBody语义|
|**内容协商**​|主要通过视图技术处理|通过HttpMessageConverter处理|
**源码层面的差异：**
```java
// @RestController的元注解定义
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Controller
@ResponseBody  // 关键差异：默认所有方法都有@ResponseBody语义
public @interface RestController {
    @AliasFor(annotation = Controller.class)
    String value() default "";
}
```
**混合使用策略：**
```java
@Controller  // 传统控制器，支持混合使用
@RequestMapping("/web")
public class WebController {
    
    // 返回视图名称 - 服务端渲染
    @GetMapping("/users")
    public String userList(Model model) {
        model.addAttribute("users", userService.findAll());
        return "user/list";  // 解析为模板页面
    }
}

@RestController  // 纯API控制器
@RequestMapping("/api")
public class ApiController {
    
    // 自动返回JSON - 前后端分离
    @GetMapping("/users")
    public List<User> getUsers() {
        return userService.findAll();  // 直接返回数据
    }
}
```
###### 9. 如何在 SpringMVC 中处理 RESTful 请求?
RESTful请求处理是Spring MVC的核心能力，通过**HTTP方法语义化**和**资源导向设计**实现规范的API设计。
**RESTful核心设计原则：**
1. **资源标识**：使用URL标识资源，如`/api/users/{id}`
2. **统一接口**：充分利用HTTP方法(GET, POST, PUT, DELETE等)
3. **无状态通信**：每次请求包含完整信息，服务器不保存会话状态
4. **超媒体驱动**：通过HATEOAS提供资源链接关系
**完整的RESTful控制器实现：**
```java
@RestController
@RequestMapping("/api/users")
public class UserRestController {
    
    @Autowired
    private UserService userService;
    
    // GET - 获取资源集合
    @GetMapping
    public ResponseEntity<List<User>> getAllUsers(
        @RequestParam(defaultValue = "0") int page,
        @RequestParam(defaultValue = "20") int size) {
        
        Page<User> users = userService.findAll(PageRequest.of(page, size));
        
        // 添加HATEOAS支持
        List<EntityModel<User>> userModels = users.stream()
            .map(user -> EntityModel.of(user,
                linkTo(methodOn(UserRestController.class).getUser(user.getId())).withSelfRel()))
            .collect(Collectors.toList());
            
        return ResponseEntity.ok()
            .header("X-Total-Count", String.valueOf(users.getTotalElements()))
            .body(userModels);
    }
    
    // GET - 获取单个资源
    @GetMapping("/{id}")
    public ResponseEntity<User> getUser(@PathVariable Long id) {
        return userService.findById(id)
            .map(user -> ResponseEntity.ok(user))
            .orElse(ResponseEntity.notFound().build());
    }
    
    // POST - 创建新资源
    @PostMapping
    public ResponseEntity<User> createUser(@Valid @RequestBody User user) {
        User savedUser = userService.save(user);
        
        URI location = ServletUriComponentsBuilder
            .fromCurrentRequest()
            .path("/{id}")
            .buildAndExpand(savedUser.getId())
            .toUri();
            
        return ResponseEntity.created(location).body(savedUser);
    }
    
    // PUT - 完整更新资源
    @PutMapping("/{id}")
    public ResponseEntity<User> updateUser(
        @PathVariable Long id, 
        @Valid @RequestBody User user) {
        
        user.setId(id);  // 确保ID一致性
        User updatedUser = userService.update(user);
        return ResponseEntity.ok(updatedUser);
    }
    
    // PATCH - 部分更新资源
    @PatchMapping("/{id}")
    public ResponseEntity<User> patchUser(
        @PathVariable Long id,
        @RequestBody Map<String, Object> updates) {
        
        User patchedUser = userService.patch(id, updates);
        return ResponseEntity.ok(patchedUser);
    }
    
    // DELETE - 删除资源
    @DeleteMapping("/{id}")
    public ResponseEntity<Void> deleteUser(@PathVariable Long id) {
        userService.delete(id);
        return ResponseEntity.noContent().build();
    }
}
```
###### 10. @RequestHeader 注解的作用是什么?
@RequestHeader用于**从HTTP请求头中提取信息**并绑定到方法参数，适用于需要访问请求头信息的场景。
**核心应用场景：**
- **用户代理检测**：读取User-Agent头进行设备适配
- **内容协商**：根据Accept头返回合适的内容类型
- **认证信息**：处理Authorization等认证头信息
- **跟踪信息**：处理Trace-Id、Session-ID等跟踪头
**高级用法示例：**
```java
@RestController
public class HeaderController {
    
    // 提取单个请求头
    @GetMapping("/info")
    public String getHeaderInfo(
        @RequestHeader("User-Agent") String userAgent,
        @RequestHeader(value = "Accept-Language", defaultValue = "zh-CN") String language) {
        return String.format("客户端: %s, 语言: %s", userAgent, language);
    }
    
    // 提取所有请求头
    @GetMapping("/headers")
    public Map<String, String> getAllHeaders(
        @RequestHeader Map<String, String> headers) {
        return headers;  // 返回所有请求头键值对
    }
    
    // 多值请求头处理
    @GetMapping("/multi")
    public String handleMultiValueHeader(
        @RequestHeader("Accept") List<String> acceptTypes) {
        return "支持的媒体类型: " + acceptTypes;
    }
    
    // 业务逻辑应用 - API版本控制
    @GetMapping("/versioned")
    public ResponseEntity<String> versionedEndpoint(
        @RequestHeader(value = "API-Version", defaultValue = "v1") String apiVersion) {
        
        switch (apiVersion) {
            case "v1":
                return ResponseEntity.ok("API版本1响应");
            case "v2":
                return ResponseEntity.ok("API版本2响应");
            default:
                return ResponseEntity.badRequest().body("不支持的API版本");
        }
    }
}
```
###### 11. @CookieValue 注解的作用是什么?
@CookieValue用于**从HTTP Cookie中提取值**并绑定到方法参数，适用于需要访问客户端Cookie信息的场景。
**Cookie处理机制：**
Spring通过`CookieMethodArgumentResolver`解析@CookieValue注解。解析器从`HttpServletRequest.getCookies()`中查找匹配的Cookie值，并进行类型转换。
**实战应用示例：**
```java
@RestController
public class CookieController {
    
    // 基础Cookie值提取
    @GetMapping("/user")
    public String getUserInfo(
        @CookieValue("sessionId") String sessionId,
        @CookieValue(value = "theme", defaultValue = "light") String theme) {
        
        User user = sessionService.getUserBySession(sessionId);
        return String.format("用户: %s, 主题: %s", user.getName(), theme);
    }
    
    // 获取完整Cookie对象
    @GetMapping("/cookie")
    public String getCookieDetails(
        @CookieValue("sessionId") Cookie sessionCookie) {
        
        return String.format("名称: %s, 值: %s, 域: %s, 路径: %s, 最大年龄: %d",
            sessionCookie.getName(),
            sessionCookie.getValue(),
            sessionCookie.getDomain(),
            sessionCookie.getPath(),
            sessionCookie.getMaxAge());
    }
    
    // 设置和删除Cookie
    @PostMapping("/login")
    public ResponseEntity<String> login(@RequestBody LoginRequest request, 
                                      HttpServletResponse response) {
        User user = authService.authenticate(request);
        String sessionId = sessionService.createSession(user);
        
        // 设置登录Cookie
        Cookie sessionCookie = new Cookie("sessionId", sessionId);
        sessionCookie.setHttpOnly(true);
        sessionCookie.setSecure(true);
        sessionCookie.setMaxAge(7 * 24 * 60 * 60);  // 7天有效期
        sessionCookie.setPath("/");
        response.addCookie(sessionCookie);
        
        return ResponseEntity.ok("登录成功");
    }
    
    @PostMapping("/logout")
    public ResponseEntity<String> logout(@CookieValue("sessionId") String sessionId,
                                       HttpServletResponse response) {
        sessionService.invalidateSession(sessionId);
        
        // 删除Cookie
        Cookie cookie = new Cookie("sessionId", "");
        cookie.setMaxAge(0);  // 立即过期
        cookie.setPath("/");
        response.addCookie(cookie);
        
        return ResponseEntity.ok("退出成功");
    }
}
```
###### 12. @ModelAttribute 注解的作用是什么?
@ModelAttribute用于**将请求参数绑定到模型对象**，支持数据预处理和模型数据管理，在表单处理中特别有用。
**工作原理：**
1. **参数绑定**：将请求参数按名称绑定到目标对象的属性
2. **数据验证**：支持JSR-303 Bean验证
3. **模型暴露**：自动将对象添加到Model中供视图使用
**多种使用方式：**
```java
@Controller
@RequestMapping("/products")
public class ProductController {
    
    // 方法级别@ModelAttribute - 在每个请求处理前执行
    @ModelAttribute("categories")
    public List<Category> loadCategories() {
        return categoryService.findAll();  // 为每个页面提供分类数据
    }
    
    // 参数级别@ModelAttribute - 表单数据绑定
    @PostMapping
    public String createProduct(@ModelAttribute("product") @Valid Product product, 
                               BindingResult result) {
        if (result.hasErrors()) {
            return "products/form";  // 返回表单页面显示错误
        }
        productService.save(product);
        return "redirect:/products";
    }
    
    // 自定义属性名绑定
    @PostMapping("/update")
    public String updateProduct(
        @ModelAttribute("prod") Product product) {  // 模型属性名为"prod"
        productService.update(product);
        return "redirect:/products";
    }
    
    // 配合@InitBinder进行自定义数据绑定
    @InitBinder
    public void initBinder(WebDataBinder binder) {
        // 自定义编辑器、验证器等
        binder.setDisallowedFields("id");  // 禁止绑定ID字段
    }
}
```
###### 13. @SessionAttributes 注解的作用是什么?
@SessionAttributes用于**在会话范围内存储模型属性**，实现跨请求的数据保持，适用于多步骤表单流程。
**会话管理机制：**
@SessionAttributes注解的控制器会将指定的模型属性自动存储到HttpSession中，并在后续请求中自动从会话恢复这些属性。
**完整的多步骤表单示例：**
```java
@Controller
@RequestMapping("/order")
@SessionAttributes("order")  // 将order属性存储到会话中
public class OrderController {
    
    // 第一步：开始创建订单
    @GetMapping("/create")
    public String createOrderForm(Model model) {
        if (!model.containsAttribute("order")) {
            model.addAttribute("order", new Order());
        }
        return "order/step1";
    }
    
    // 处理第一步数据
    @PostMapping("/step1")
    public String processStep1(
        @ModelAttribute("order") @Valid Order order,
        BindingResult result) {
        
        if (result.hasErrors()) {
            return "order/step1";
        }
        return "redirect:/order/step2";
    }
    
    // 第二步：确认订单
    @GetMapping("/step2")
    public String step2Form() {
        return "order/step2";
    }
    
    // 处理第二步数据
    @PostMapping("/step2")
    public String processStep2(
        @ModelAttribute("order") @Valid Order order,
        BindingResult result) {
        
        if (result.hasErrors()) {
            return "order/step2";
        }
        return "redirect:/order/confirm";
    }
    
    // 完成订单，清理会话属性
    @PostMapping("/confirm")
    public String confirmOrder(
        @ModelAttribute("order") Order order,
        SessionStatus status) {
        
        Order savedOrder = orderService.save(order);
        status.setComplete();  // 标记会话处理完成，清理会话属性
        
        return "redirect:/order/complete/" + savedOrder.getId();
    }
    
    // 完成页面
    @GetMapping("/complete/{id}")
    public String orderComplete(@PathVariable Long id, Model model) {
        model.addAttribute("order", orderService.findById(id));
        return "order/complete";
    }
}
```