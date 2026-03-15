###### 1. 什么是 SpringMVC？

SpringMVC 是 Spring 框架的 Web 层模块，基于 **MVC（Model-View-Controller）设计模式**，用于构建 Web 应用程序。它以 `DispatcherServlet` 作为前端控制器，统一拦截 HTTP 请求，协调各组件完成请求处理。

简单说，SpringMVC 帮你把一个 HTTP 请求路由到正确的处理方法，处理结果转成响应返回客户端，中间的所有繁琐工作（参数解析、类型转换、视图渲染）都由框架负责。

它的核心价值在于：与 Spring IoC/AOP 无缝集成，POJO 化开发（不强制继承框架类），注解驱动配置，天然支持 RESTful 风格，支持多种视图技术（JSP、Thymeleaf、FreeMarker）。

📖 [[../../../../25_SpringMVCKnowledge/01_核心架构与流程/01、SpringMVC核心架构与请求处理流程]]

---

###### 2. SpringMVC 的核心组件有哪些？

SpringMVC 的架构围绕以下几个核心组件构建：

**DispatcherServlet（前端控制器）**：整个流程的调度中心，是标准 Java Servlet，所有请求都经过它。核心逻辑在 `doDispatch()` 方法。

**HandlerMapping（处理器映射器）**：负责建立 URL → Handler 的映射关系，最常用的是 `RequestMappingHandlerMapping`，它解析 `@RequestMapping` 注解并维护映射表。

**HandlerAdapter（处理器适配器）**：屏蔽不同类型 Handler 的调用差异，让 DispatcherServlet 以统一方式调用控制器方法，内部负责参数解析、反射调用、返回值处理。应用了适配器设计模式。

**ViewResolver（视图解析器）**：把逻辑视图名（如 `"user/list"`）解析为具体的 `View` 对象，支持多种视图技术。应用了策略设计模式。

**HandlerInterceptor（拦截器）**：提供 `preHandle`/`postHandle`/`afterCompletion` 三个生命周期回调，实现权限验证、日志、性能监控等横切功能。

📖 [[../../../../25_SpringMVCKnowledge/01_核心架构与流程/01、SpringMVC核心架构与请求处理流程#核心组件]]

---

###### 3. SpringMVC 的优点有哪些？

主要优点包括：与 Spring 生态系统无缝集成（IoC、AOP、声明式事务）；POJO 化设计，低耦合非侵入式；强大的数据绑定和校验（JSR-303 Bean Validation）；卓越的 RESTful 支持（`@RestController` + HTTP 方法注解）；可插拔的视图技术；接口驱动设计，扩展性强（自定义 HandlerMapping/HandlerAdapter/ArgumentResolver 等）；注解 + Java Config 驱动，配置简洁。

📖 [[../../../../25_SpringMVCKnowledge/01_核心架构与流程/01、SpringMVC核心架构与请求处理流程]]

---

###### 4. SpringMVC 的工作原理是什么？

一次请求从到达到响应的完整流程：

1. HTTP 请求到达，被 Tomcat 转发给 `DispatcherServlet`
2. `DispatcherServlet` 调用 `HandlerMapping.getHandler()`，查找处理器执行链（Handler + 拦截器列表）
3. 依次执行拦截器的 `preHandle()`，返回 false 则中断请求
4. 获取 `HandlerAdapter`，通过适配器调用 Handler（参数解析 → 反射调用 → 返回值处理）
5. 执行拦截器的 `postHandle()`
6. 处理渲染：有 `@ResponseBody` 直接写响应体；否则通过 `ViewResolver` 解析视图名并渲染
7. 执行拦截器的 `afterCompletion()`（类似 finally，异常也执行）
8. 返回响应

📖 [[../../../../25_SpringMVCKnowledge/01_核心架构与流程/01、SpringMVC核心架构与请求处理流程#请求处理完整流程]]
