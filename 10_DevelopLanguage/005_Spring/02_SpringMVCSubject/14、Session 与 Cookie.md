###### 1. SpringMVC 如何操作 Session？

SpringMVC 提供了多种 Session 操作方式，从底层 Servlet API 到声明式的注解方案，按场景选用：

**方式一：直接注入 `HttpSession`（最直接）**

```java
@Controller
public class SessionController {

    @PostMapping("/login")
    public String login(HttpServletRequest request, @RequestParam String username) {
        HttpSession session = request.getSession(true);  // true 表示不存在则创建
        session.setAttribute("currentUser", username);
        session.setMaxInactiveInterval(30 * 60);  // 30 分钟超时
        return "redirect:/dashboard";
    }

    // 也可以直接注入 HttpSession 参数（Spring 自动调用 request.getSession()）
    @GetMapping("/profile")
    public String getProfile(HttpSession session, Model model) {
        String user = (String) session.getAttribute("currentUser");
        if (user == null) return "redirect:/login";
        model.addAttribute("user", user);
        return "profile";
    }

    @PostMapping("/logout")
    public String logout(HttpSession session) {
        session.invalidate();  // 使整个 Session 失效
        return "redirect:/login";
    }
}
```

**方式二：`@SessionAttributes` 声明式管理（控制器内多步骤场景）**

```java
@Controller
@SessionAttributes({"user", "cart"})  // 声明需要跨请求保持的模型属性
public class UserController {

    @PostMapping("/user/login")
    public String login(@Valid User user, Model model) {
        model.addAttribute("user", user);  // 自动存入 Session
        model.addAttribute("cart", new ShoppingCart());
        return "redirect:/home";
    }

    @PostMapping("/user/complete")
    public String completeOrder(SessionStatus status) {
        status.setComplete();  // 清除 @SessionAttributes 管理的 Session 属性
        return "order-complete";
    }
}
```

📖 [[25_SpringMVCKnowledge/07_配置扩展与整合/01、配置扩展与技术整合]]

---

###### 2. SpringMVC 如何操作 Cookie？

Cookie 的读取推荐用 `@CookieValue` 注解，创建和删除则通过 `HttpServletResponse` 操作：

**创建 Cookie**

```java
@Controller
public class CookieController {

    @PostMapping("/preferences")
    public String setPreferences(
        @RequestParam String theme,
        HttpServletResponse response) {

        Cookie themeCookie = new Cookie("app_theme", theme);
        themeCookie.setMaxAge(30 * 24 * 60 * 60);  // 30 天有效期
        themeCookie.setPath("/");
        themeCookie.setHttpOnly(true);   // 禁止 JavaScript 读取，防 XSS
        themeCookie.setSecure(true);     // 只通过 HTTPS 传输，防中间人
        response.addCookie(themeCookie);

        return "redirect:/settings";
    }
}
```

**读取 Cookie（`@CookieValue` 注解）**

```java
@RestController
public class PreferenceController {

    @GetMapping("/api/preferences")
    public Map<String, String> getPreferences(
        @CookieValue(value = "app_theme", defaultValue = "light") String theme,
        @CookieValue(value = "user_lang", defaultValue = "zh-CN") String language) {

        return Map.of("theme", theme, "language", language);
    }
}
```

**删除 Cookie**（把 `maxAge` 设为 0，让浏览器立即过期）

```java
@PostMapping("/clear-preferences")
public String clearCookies(HttpServletResponse response) {
    Cookie cookie = new Cookie("app_theme", "");
    cookie.setMaxAge(0);   // 立即过期
    cookie.setPath("/");   // Path 要和创建时一致，否则删不掉！
    response.addCookie(cookie);
    return "redirect:/";
}
```

删除 Cookie 时，`Path` 和 `Domain` 必须和创建时完全一致，否则浏览器会认为是两个不同的 Cookie，删除不会生效。

📖 [[25_SpringMVCKnowledge/07_配置扩展与整合/01、配置扩展与技术整合]]

---

###### 3. @SessionAttribute 注解的作用是什么？

`@SessionAttribute` 是方法参数级注解，用于从 Session 中直接提取已存在的属性值注入到控制器方法，适合在不同请求间共享 Session 中的用户信息。

注意和 `@SessionAttributes`（类级别，用于声明哪些 Model 属性存 Session）区分开：

- `@SessionAttributes`：类级别，声明 Model 属性同步到 Session 的策略
- `@SessionAttribute`：方法参数级别，从 Session 中取出值注入参数

```java
@Controller
public class UserDashboardController {

    // 从 Session 中取 currentUser（required 默认为 true，不存在会抛异常）
    @GetMapping("/dashboard")
    public String showDashboard(
        @SessionAttribute("currentUser") User user,
        Model model) {
        model.addAttribute("userName", user.getName());
        return "dashboard";
    }

    // required = false：Session 中没有时注入 null，不抛异常
    @GetMapping("/profile")
    public String showProfile(
        @SessionAttribute(value = "userPrefs", required = false) Preferences prefs,
        Model model) {
        model.addAttribute("preferences",
            prefs != null ? prefs : Preferences.getDefaults());
        return "profile";
    }
}
```

📖 [[25_SpringMVCKnowledge/07_配置扩展与整合/01、配置扩展与技术整合]]

---

###### 4. HttpSession 和 @SessionAttributes 的区别是什么？

两者都是 Session 管理工具，但面向不同的场景：

**`HttpSession`**：Servlet 规范标准 API，**全局**可用（Service 层、任意控制器都能访问），需要**显式**操作（`setAttribute`/`getAttribute`/`invalidate`），适合需要精确控制生命周期的场景，如跨模块的用户状态、购物车等。

**`@SessionAttributes`**：Spring MVC 框架特性，**控制器级别**隔离（只在当前控制器内有效，其他控制器访问不到），声明式地把 Model 属性自动同步到 Session，用 `SessionStatus.setComplete()` 清理，适合控制器内的多步骤表单流程。

```java
// HttpSession：全局，任意组件可访问
@Service
public class CartService {
    public void addToCart(HttpSession session, Product product) {
        ShoppingCart cart = (ShoppingCart) session.getAttribute("cart");
        if (cart == null) cart = new ShoppingCart();
        cart.addItem(product);
        session.setAttribute("cart", cart);
    }
}

// @SessionAttributes：控制器内隔离，多步骤表单
@Controller
@SessionAttributes("orderForm")  // 只在 OrderFormController 内有效
public class OrderFormController {

    @GetMapping("/order/step1")
    public String step1(Model model) {
        if (!model.containsAttribute("orderForm")) {
            model.addAttribute("orderForm", new OrderForm());
        }
        return "step1";
    }

    @PostMapping("/order/complete")
    public String complete(
        @ModelAttribute("orderForm") OrderForm form,
        SessionStatus status) {
        orderService.submit(form);
        status.setComplete();  // 清理 Session 中的 orderForm
        return "redirect:/success";
    }
}
```

实际项目里两者经常配合使用：`@SessionAttributes` 处理向导式表单，`HttpSession` 处理用户登录状态等全局数据。

📖 [[25_SpringMVCKnowledge/07_配置扩展与整合/01、配置扩展与技术整合]]
