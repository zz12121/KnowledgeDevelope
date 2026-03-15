# SpringMVC 核心架构与工作流程

## SpringMVC 是什么

SpringMVC 是 Spring 框架的 Web 层模块，基于 **MVC（Model-View-Controller）设计模式**，用于构建 Web 应用程序。它以 `DispatcherServlet` 作为前端控制器，统一拦截 HTTP 请求，协调各组件完成请求处理。

核心定位：与 Spring 生态（IoC、AOP、事务）无缝集成，POJO 化设计（低耦合），注解驱动，支持多种视图技术，支持 RESTful 风格。

---

## 核心组件

SpringMVC 的核心架构由以下几个组件构成：

**DispatcherServlet（前端控制器）**：整个流程的调度中心，是标准 Java Servlet，实现前端控制器模式。所有请求经过它分发，核心逻辑在 `doDispatch()` 方法中。

**HandlerMapping（处理器映射器）**：根据 URL、HTTP 方法找到对应处理器，最常用的是 `RequestMappingHandlerMapping`，负责解析 `@RequestMapping` 注解。

**HandlerAdapter（处理器适配器）**：屏蔽不同类型处理器的调用差异，使 DispatcherServlet 能以统一方式调用。最常用的是 `RequestMappingHandlerAdapter`，负责参数解析、方法反射调用、返回值处理。

**ViewResolver（视图解析器）**：将控制器返回的逻辑视图名解析为具体的 View 对象，策略模式设计，支持多种视图技术（JSP、Thymeleaf、FreeMarker）。

**View（视图）**：负责渲染模型数据，生成最终 HTTP 响应内容。

**HandlerInterceptor（拦截器）**：提供三个生命周期回调（`preHandle`/`postHandle`/`afterCompletion`），实现权限验证、日志、性能监控等横切功能。

---

## 请求处理完整流程

一次 HTTP 请求在 SpringMVC 中的完整旅程：

1. HTTP 请求到达，被 Tomcat 转发给 `DispatcherServlet`
2. `DispatcherServlet` 调用 `HandlerMapping.getHandler()`，查找处理器执行链（包含 Handler + 拦截器列表）
3. 遍历拦截器，依次执行 `preHandle()`，返回 false 则中断请求
4. 获取对应的 `HandlerAdapter`，通过适配器调用 Handler（参数解析 → 反射调用 → 返回值处理）
5. 执行拦截器的 `postHandle()`
6. 处理渲染结果：`@ResponseBody` 直接写响应体；否则通过 `ViewResolver` 解析视图名，调用 `View.render()` 渲染
7. 执行拦截器的 `afterCompletion()`（类似 finally 块，异常也会执行）
8. 返回响应

---

## DispatcherServlet 初始化

`DispatcherServlet` 继承体系：`HttpServlet` → `HttpServletBean` → `FrameworkServlet` → `DispatcherServlet`。

初始化时调用 `initStrategies()` 方法，依次初始化九大组件：

```java
protected void initStrategies(ApplicationContext context) {
    initMultipartResolver(context);      // 文件上传解析器
    initLocaleResolver(context);         // 本地化解析器
    initThemeResolver(context);          // 主题解析器
    initHandlerMappings(context);        // 处理器映射器
    initHandlerAdapters(context);        // 处理器适配器
    initHandlerExceptionResolvers(context); // 异常解析器
    initRequestToViewNameTranslator(context); // 视图名转换器
    initViewResolvers(context);          // 视图解析器
    initFlashMapManager(context);        // Flash属性管理器
}
```

每个组件都有默认实现，也可以在容器中注册自定义 Bean 来覆盖。

---

## 九大组件一览

| 组件 | 作用 |
|------|------|
| `MultipartResolver` | 文件上传解析器，检测 multipart 请求并封装为 `MultipartFile` |
| `LocaleResolver` | 语言环境解析器，决定当前请求使用哪个 Locale |
| `ThemeResolver` | 主题解析器，支持多套 UI 主题切换 |
| `HandlerMapping` | 处理器映射器，URL → Handler |
| `HandlerAdapter` | 处理器适配器，统一调用不同类型的 Handler |
| `HandlerExceptionResolver` | 异常解析器，统一处理控制器抛出的异常 |
| `RequestToViewNameTranslator` | 视图名转换器，从请求路径推导视图名 |
| `ViewResolver` | 视图解析器，逻辑视图名 → View 对象 |
| `FlashMapManager` | Flash 属性管理器，重定向时传递临时数据 |

---

## 相关面试题 →

[[../../10_Developlanguage/005_Spring/02_SpringMVCSubject/01、SpringMVC 基础概念]]
[[../../10_Developlanguage/005_Spring/02_SpringMVCSubject/02、DispatcherServlet 与请求处理流程]]
[[../../10_Developlanguage/005_Spring/02_SpringMVCSubject/19、源码与原理]]
