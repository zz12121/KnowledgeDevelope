###### 1. SpringMVC 如何操作 Session?
SpringMVC提供了**多层次、多方式的Session操作机制**，从底层的Servlet API到高级的注解驱动方案，满足不同场景下的会话管理需求。
核心操作方式分析
**方式一：基于Servlet API的直接操作**（最基础方案）
通过方法参数注入`HttpServletRequest`或直接注入`HttpSession`对象，使用标准的Servlet API进行Session操作。
```java
@Controller
public class SessionController {
    
    // 通过HttpServletRequest获取Session
    @PostMapping("/login")
    public String login(HttpServletRequest request, @RequestParam String username) {
        HttpSession session = request.getSession();
        session.setAttribute("user", username);
        session.setMaxInactiveInterval(30 * 60); // 30分钟超时
        return "dashboard";
    }
    
    // 直接注入HttpSession（Spring MVC自动调用request.getSession()）
    @GetMapping("/profile")
    public String getProfile(HttpSession session) {
        String user = (String) session.getAttribute("user");
        if (user == null) {
            return "redirect:/login";
        }
        return "profile";
    }
    
    // Session失效管理
    @PostMapping("/logout")
    public String logout(HttpSession session) {
        session.removeAttribute("user");  // 移除特定属性
        session.invalidate();             // 使整个Session失效
        return "redirect:/login";
    }
}
```
**方式二：基于`@SessionAttributes`的声明式管理**（Spring特色方案）
在控制器类级别使用`@SessionAttributes`注解，自动将模型属性存储到Session中。
```java
@Controller
@SessionAttributes({"user", "cart"})  // 声明需要Session化的模型属性
public class UserController {
    
    @PostMapping("/user/login")
    public String login(@Valid User user, Model model) {
        // 模型属性自动存储到Session（因为类有@SessionAttributes注解）
        model.addAttribute("user", user);
        model.addAttribute("cart", new ShoppingCart());
        return "redirect:/home";
    }
    
    @GetMapping("/user/cart")
    public String viewCart(@ModelAttribute("cart") ShoppingCart cart) {
        // 自动从Session中获取cart对象
        return "cart";
    }
    
    @PostMapping("/user/complete")
    public String completeOrder(SessionStatus status) {
        status.setComplete();  // 清除通过@SessionAttributes存储的Session属性
        return "order-complete";
    }
}
```
源码级机制解析
Spring MVC通过`DispatcherServlet`的统一请求处理流程集成Session管理。在`doDispatch()`方法中，当控制器方法需要`HttpSession`参数时，`RequestMappingHandlerAdapter`会使用`HttpSessionMethodArgumentResolver`来解析参数。
```java
// Spring MVC参数解析的核心机制
public class RequestMappingHandlerAdapter extends AbstractHandlerMethodAdapter {
    protected ModelAndView invokeHandlerMethod(HttpServletRequest request,
                                              HttpServletResponse response,
                                              HandlerMethod handlerMethod) throws Exception {
        // 设置参数解析器，包含Session相关的解析器
        ServletInvocableHandlerMethod invocableMethod = createInvocableHandlerMethod(handlerMethod);
        invocableMethod.setHandlerMethodArgumentResolvers(this.argumentResolvers);
        
        // 执行方法调用
        invocableMethod.invokeAndHandle(webRequest, mavContainer);
        return getModelAndView(mavContainer, modelFactory, webRequest);
    }
}

// HttpSession参数解析器
public class HttpSessionMethodArgumentResolver implements HandlerMethodArgumentResolver {
    public boolean supportsParameter(MethodParameter parameter) {
        return HttpSession.class.isAssignableFrom(parameter.getParameterType());
    }
    
    public Object resolveArgument(MethodParameter parameter, ModelAndViewContainer mavContainer,
                                  NativeWebRequest request, WebDataBinderFactory binderFactory) throws Exception {
        // 直接从NativeWebRequest中获取HttpSession
        return request.getNativeRequest(HttpServletRequest.class).getSession();
    }
}
```
###### 2. SpringMVC 如何操作 Cookie?
SpringMVC提供**完整的Cookie操作支持**，包括Cookie的创建、读取、修改和删除，同时通过`@CookieValue`注解简化数据获取。
完整Cookie操作示例
**Cookie的创建与配置**
```java
@Controller
public class CookieController {
    
    @PostMapping("/preferences")
    public String setPreferences(@RequestParam String theme, 
                                @RequestParam String language,
                                HttpServletResponse response) {
        // 创建主题Cookie
        Cookie themeCookie = new Cookie("app_theme", theme);
        themeCookie.setMaxAge(30 * 24 * 60 * 60); // 30天有效期
        themeCookie.setPath("/");
        themeCookie.setHttpOnly(true);    // 防XSS
        themeCookie.setSecure(true);      // 仅HTTPS传输
        response.addCookie(themeCookie);
        
        // 创建语言Cookie
        Cookie langCookie = new Cookie("user_lang", language);
        langCookie.setMaxAge(30 * 24 * 60 * 60);
        langCookie.setPath("/");
        response.addCookie(langCookie);
        
        return "redirect:/settings";
    }
}
```
**使用`@CookieValue`注解简化读取**
```java
@RestController
public class PreferenceController {
    
    @GetMapping("/api/preferences")
    public ResponseEntity<Map<String, String>> getPreferences(
            @CookieValue(value = "app_theme", defaultValue = "light") String theme,
            @CookieValue(value = "user_lang", defaultValue = "zh-CN") String language) {
        
        Map<String, String> prefs = new HashMap<>();
        prefs.put("theme", theme);
        prefs.put("language", language);
        return ResponseEntity.ok(prefs);
    }
}
```
**Cookie的更新与删除**
```java
@Controller
public class CookieManagementController {
    
    @PostMapping("/update-theme")
    public String updateTheme(@RequestParam String newTheme, 
                            HttpServletRequest request,
                            HttpServletResponse response) {
        // 查找现有Cookie并更新
        Cookie[] cookies = request.getCookies();
        if (cookies != null) {
            for (Cookie cookie : cookies) {
                if ("app_theme".equals(cookie.getName())) {
                    cookie.setValue(newTheme);
                    cookie.setMaxAge(30 * 24 * 60 * 60); // 重置有效期
                    response.addCookie(cookie);
                    break;
                }
            }
        }
        return "redirect:/preferences";
    }
    
    @PostMapping("/clear-cookies")
    public String clearCookies(HttpServletResponse response) {
        // 删除Cookie：设置maxAge为0
        Cookie themeCookie = new Cookie("app_theme", "");
        themeCookie.setMaxAge(0);
        themeCookie.setPath("/");
        response.addCookie(themeCookie);
        
        Cookie langCookie = new Cookie("user_lang", "");
        langCookie.setMaxAge(0);
        langCookie.setPath("/");
        response.addCookie(langCookie);
        
        return "redirect:/login";
    }
}
```
源码机制：`@CookieValue`注解处理
Spring MVC通过`CookieMethodArgumentResolver`处理`@CookieValue`注解，该解析器实现了`HandlerMethodArgumentResolver`接口：
```java
public class CookieMethodArgumentResolver implements HandlerMethodArgumentResolver {
    
    public boolean supportsParameter(MethodParameter parameter) {
        return parameter.hasParameterAnnotation(CookieValue.class);
    }
    
    public Object resolveArgument(MethodParameter parameter, 
                                  ModelAndViewContainer mavContainer,
                                  NativeWebRequest webRequest, 
                                  WebDataBinderFactory binderFactory) throws Exception {
        
        CookieValue annotation = parameter.getParameterAnnotation(CookieValue.class);
        String name = annotation.value();
        boolean required = annotation.required();
        String defaultValue = annotation.defaultValue();
        
        // 从请求中获取Cookie数组
        Cookie[] cookies = webRequest.getNativeRequest(HttpServletRequest.class).getCookies();
        String value = null;
        
        if (cookies != null) {
            for (Cookie cookie : cookies) {
                if (name.equals(cookie.getName())) {
                    value = cookie.getValue();
                    break;
                }
            }
        }
        
        if (value == null) {
            if (required && defaultValue.equals(ValueConstants.DEFAULT_NONE)) {
                throw new ServletRequestBindingException("Missing cookie '" + name + "'");
            }
            value = defaultValue;
        }
        
        // 类型转换
        return binderFactory.createDataBinder(webRequest, value, name)
                           .convertIfNecessary(value, parameter.getParameterType());
    }
}
```
###### 3. @SessionAttribute 注解的作用是什么?
`@SessionAttribute`是一个**方法参数级别的注解**，用于从Session中直接提取已存在的属性值并注入到控制器方法参数中。
核心功能与使用场景
**基本用法示例**
```java
@Controller
public class UserDashboardController {
    
    // 从Session中提取用户信息
    @GetMapping("/dashboard")
    public String showDashboard(@SessionAttribute("currentUser") User user, Model model) {
        model.addAttribute("userName", user.getName());
        model.addAttribute("role", user.getRole());
        return "dashboard";
    }
    
    // 带默认值和可选配置
    @GetMapping("/profile")
    public String showProfile(@SessionAttribute(value = "userPrefs", required = false) 
                            Preferences prefs, Model model) {
        if (prefs == null) {
            prefs = Preferences.getDefaults(); // 使用默认配置
        }
        model.addAttribute("preferences", prefs);
        return "profile";
    }
}
```
**高级特性：类型安全与验证**
```java
@RestController
public class ApiController {
    
    @GetMapping("/api/user/info")
    public ResponseEntity<UserInfo> getUserInfo(
            @SessionAttribute(value = "currentUser", required = true) User user) {
        // 由于required=true，如果Session中没有currentUser会抛出异常
        UserInfo info = userService.getUserInfo(user.getId());
        return ResponseEntity.ok(info);
    }
    
    // 使用Java 8 Optional避免空指针
    @GetMapping("/api/settings")
    public ResponseEntity<Settings> getSettings(
            @SessionAttribute(value = "userSettings", required = false) 
            Optional<Settings> settingsOpt) {
        
        Settings settings = settingsOpt.orElse(Settings.getDefault());
        return ResponseEntity.ok(settings);
    }
}
```
源码解析：SessionAttributeMethodArgumentResolver
`@SessionAttribute`注解由`SessionAttributeMethodArgumentResolver`处理，其核心解析逻辑如下：
```java
public class SessionAttributeMethodArgumentResolver implements HandlerMethodArgumentResolver {
    
    public boolean supportsParameter(MethodParameter parameter) {
        return parameter.hasParameterAnnotation(SessionAttribute.class);
    }
    
    public Object resolveArgument(MethodParameter parameter, 
                                  ModelAndViewContainer mavContainer,
                                  NativeWebRequest webRequest, 
                                  WebDataBinderFactory binderFactory) throws Exception {
        
        SessionAttribute ann = parameter.getParameterAnnotation(SessionAttribute.class);
        String name = ann.name().isEmpty() ? parameter.getParameterName() : ann.name();
        boolean required = ann.required();
        
        // 从Session作用域获取属性
        Object value = webRequest.getAttribute(name, RequestAttributes.SCOPE_SESSION);
        
        if (value == null && required) {
            throw new ServletRequestBindingException("Missing session attribute '" + name + "'");
        }
        
        return value;
    }
}
```
###### 4. HttpSession 和 @SessionAttributes 的区别是什么?
`HttpSession`和`@SessionAttributes`是Spring MVC中**两种不同层次、不同设计目标的Session管理机制**，理解它们的区别对正确选择会话方案至关重要。
核心差异对比

|**特性维度**​|**HttpSession**​|**@SessionAttributes**​|
|---|---|---|
|**设计层级**​|Servlet规范标准API|Spring MVC框架特性|
|**作用范围**​|应用全局（所有控制器、服务层）|当前控制器类内部|
|**数据来源**​|直接操作Session属性|自动将Model属性同步到Session|
|**生命周期**​|显式管理（set/remove/invalidate）|与Model生命周期绑定，可通过SessionStatus清除|
|**使用场景**​|全局用户状态、跨控制器数据共享|控制器内多步骤表单、向导式操作|
技术细节深度对比
**HttpSession的全局性特点**
```java
// Service层中使用HttpSession（跨控制器共享）
@Service
public class ShoppingCartService {
    public void addToCart(HttpSession session, Product product) {
        // 任意组件都可以访问Session
        ShoppingCart cart = (ShoppingCart) session.getAttribute("cart");
        if (cart == null) {
            cart = new ShoppingCart();
            session.setAttribute("cart", cart);
        }
        cart.addItem(product);
    }
}

// 在另一个控制器中访问同一Session属性
@Controller
public class OrderController {
    @PostMapping("/checkout")
    public String checkout(HttpSession session) {
        // 可以访问其他组件设置的Session属性
        ShoppingCart cart = (ShoppingCart) session.getAttribute("cart");
        if (cart == null || cart.isEmpty()) {
            return "redirect:/cart";
        }
        return "checkout";
    }
}
```
**@SessionAttributes的控制器隔离性**
```java
@Controller
@SessionAttributes("formData")  // 仅在此控制器内有效
public class MultiStepFormController {
    
    @GetMapping("/form/step1")
    public String showStep1(Model model) {
        if (!model.containsAttribute("formData")) {
            model.addAttribute("formData", new FormData());
        }
        return "step1";
    }
    
    @PostMapping("/form/step1")
    public String processStep1(@ModelAttribute("formData") FormData formData) {
        // formData自动在Step间通过Session传递
        return "redirect:/form/step2";
    }
    
    @GetMapping("/form/step2")
    public String showStep2(@ModelAttribute("formData") FormData formData) {
        return "step2";
    }
    
    @PostMapping("/form/complete")
    public String completeForm(@ModelAttribute("formData") FormData formData, 
                              SessionStatus status) {
        service.processForm(formData);
        status.setComplete();  // 清除@SessionAttributes管理的属性
        return "redirect:/success";
    }
}

// 其他控制器无法访问上述formData
@Controller
public class OtherController {
    @GetMapping("/other")
    public String otherMethod(HttpSession session) {
        // 这里获取不到"formData"，因为@SessionAttributes是控制器级别的
        Object formData = session.getAttribute("formData"); // 可能为null
        return "other";
    }
}
```
源码级架构差异
**HttpSession的直接存储机制**
```java
// Servlet规范层面的直接操作
public interface HttpSession {
    void setAttribute(String name, Object value);
    Object getAttribute(String name);
    void removeAttribute(String name);
}
```
**@SessionAttributes的代理机制**
Spring MVC通过`SessionAttributesHandler`管理`@SessionAttributes`注解声明的属性，在`RequestMappingHandlerAdapter`中集成：
```java
public class RequestMappingHandlerAdapter extends AbstractHandlerMethodAdapter {
    protected ModelAndView invokeHandlerMethod(HttpServletRequest request,
                                              HttpServletResponse response,
                                              HandlerMethod handlerMethod) throws Exception {
        
        ServletInvocableHandlerMethod invocableMethod = createInvocableHandlerMethod(handlerMethod);
        
        // 设置SessionAttributes处理器
        SessionAttributesHandler sessionAttributesHandler = getSessionAttributesHandler(handlerMethod);
        invocableMethod.setSessionAttributesHandler(sessionAttributesHandler);
        
        // 处理过程中自动管理Session属性同步
        invocableMethod.invokeAndHandle(webRequest, mavContainer);
    }
}
```
**设计价值总结**：`HttpSession`提供**全局、显式、底层**的Session控制，适合需要精确管理的复杂场景；`@SessionAttributes`提供**控制器内、声明式、自动化**的Session同步，适合简单的多步骤数据流。在实际项目中，两者常结合使用，发挥各自优势。