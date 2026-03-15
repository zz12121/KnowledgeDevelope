###### 1. 什么是 DispatcherServlet？

`DispatcherServlet` 是 SpringMVC 中**前端控制器模式**的具体实现，是一个标准 Java Servlet，作为 Spring Web 应用的统一入口点。

它的核心特征是：**集中控制点**，所有请求都经过它统一分发；**与 Spring IoC 无缝集成**，每个 DispatcherServlet 都有对应的 `WebApplicationContext`；**可配置的策略接口**，九大组件都可以替换为自定义实现。

继承体系：`HttpServlet` → `HttpServletBean` → `FrameworkServlet` → `DispatcherServlet`。

📖 [[../../../../25_SpringMVCKnowledge/01_核心架构与流程/01、SpringMVC核心架构与请求处理流程#DispatcherServlet初始化]]

---

###### 2. DispatcherServlet 的作用是什么？

核心作用是**请求分发与协调**：接收所有 HTTP 请求，协调 HandlerMapping、HandlerAdapter、ViewResolver 等组件完成请求处理流程。

初始化时调用 `initStrategies()` 方法，依次初始化九大组件：

```java
protected void initStrategies(ApplicationContext context) {
    initMultipartResolver(context);      // 文件上传
    initLocaleResolver(context);         // 国际化
    initThemeResolver(context);          // 主题
    initHandlerMappings(context);        // 处理器映射
    initHandlerAdapters(context);        // 处理器适配
    initHandlerExceptionResolvers(context); // 异常处理
    initRequestToViewNameTranslator(context); // 视图名转换
    initViewResolvers(context);          // 视图解析
    initFlashMapManager(context);        // 重定向数据传递
}
```

每个组件都会先从容器中查找自定义 Bean，找不到才使用默认实现（定义在 `DispatcherServlet.properties` 中）。

📖 [[../../../../25_SpringMVCKnowledge/01_核心架构与流程/01、SpringMVC核心架构与请求处理流程#九大组件一览]]

---

###### 3. SpringMVC 的请求处理流程是怎样的？

见第一个文档第 4 题，流程相同。核心方法是 `doDispatch()`，伪代码：

```java
protected void doDispatch(HttpServletRequest request, HttpServletResponse response) {
    // 1. 检查是否 multipart 请求
    processedRequest = checkMultipart(request);
    
    // 2. 获取处理器执行链（handler + interceptors）
    HandlerExecutionChain mappedHandler = getHandler(processedRequest);
    
    // 3. 获取适配器
    HandlerAdapter ha = getHandlerAdapter(mappedHandler.getHandler());
    
    // 4. 执行前置拦截器
    mappedHandler.applyPreHandle(processedRequest, response);
    
    // 5. 执行处理器
    ModelAndView mv = ha.handle(processedRequest, response, mappedHandler.getHandler());
    
    // 6. 执行后置拦截器
    mappedHandler.applyPostHandle(processedRequest, response, mv);
    
    // 7. 处理渲染结果
    processDispatchResult(processedRequest, response, mappedHandler, mv, ...);
}
```

📖 [[../../../../25_SpringMVCKnowledge/01_核心架构与流程/01、SpringMVC核心架构与请求处理流程#请求处理完整流程]]

---

###### 4. 前端控制器的作用是什么？

前端控制器模式把所有请求的接收和分发集中到一个入口，好处是：

**集中处理横切关注点**：认证、日志、国际化、异常处理，都在这一个地方统一搞定，不用每个控制器重复写。

**请求处理流程标准化**：所有请求都走相同的预处理 → 核心处理 → 后处理流程，一致且可预期。

**组件解耦**：Handler 只负责业务逻辑，HandlerMapping 负责路由，ViewResolver 负责视图，各司其职，互相不知道。

没有前端控制器的话，每个 Servlet 都要自己处理路由、参数解析、异常，代码重复且难以维护。DispatcherServlet 把这些通用逻辑集中到一处，大大简化了开发。

📖 [[../../../../25_SpringMVCKnowledge/01_核心架构与流程/01、SpringMVC核心架构与请求处理流程]]

---

###### 5. HandlerMapping 的作用是什么？

`HandlerMapping` 负责**建立 HTTP 请求与处理器之间的映射关系**，核心接口只有一个方法：`getHandler(HttpServletRequest) → HandlerExecutionChain`。

`RequestMappingHandlerMapping` 是最常用的实现，工作分两阶段：
- **初始化**：扫描所有 `@Controller` Bean，找到所有 `@RequestMapping` 方法，注册到映射表
- **运行时**：根据请求 URL、HTTP 方法等信息查找最匹配的 `HandlerMethod`，支持路径变量匹配（`/users/{id}`）和通配符匹配

返回的 `HandlerExecutionChain` 除了 Handler 本身，还包含了所有匹配的拦截器，DispatcherServlet 直接用这个链路执行。

📖 [[../../../../25_SpringMVCKnowledge/08_性能优化与源码原理/01、SpringMVC性能优化与源码原理#HandlerMapping源码原理]]

---

###### 6. HandlerAdapter 的作用是什么？

`HandlerAdapter` 屏蔽了不同类型处理器（`@RequestMapping` 方法、`Controller` 接口、`HttpRequestHandler` 接口）的调用差异，让 DispatcherServlet 用统一的接口调用所有类型的 Handler，是典型的适配器设计模式。

`RequestMappingHandlerAdapter` 核心执行流程：
1. 通过 `HandlerMethodArgumentResolver` 解析方法参数
2. 通过反射调用目标方法
3. 通过 `HandlerMethodReturnValueHandler` 处理返回值

📖 [[../../../../25_SpringMVCKnowledge/08_性能优化与源码原理/01、SpringMVC性能优化与源码原理#HandlerAdapter源码原理]]

---

###### 7. ViewResolver 的作用是什么？

`ViewResolver` 把控制器返回的逻辑视图名（比如 `"user/list"`）解析为具体的 `View` 实现对象，再由 `View.render()` 渲染模型数据生成 HTML/JSON 等响应。

可以配置多个 ViewResolver，通过 `order` 属性控制优先级，DispatcherServlet 遍历所有 ViewResolver，第一个能解析的就用它。

常用实现：`InternalResourceViewResolver`（JSP）、`ThymeleafViewResolver`（Thymeleaf）、`FreeMarkerViewResolver`（FreeMarker）。

前后端分离项目通常不需要 ViewResolver，因为 `@ResponseBody` 直接把结果序列化到响应体，绕过了视图解析流程。

📖 [[../../../../25_SpringMVCKnowledge/04_视图解析与渲染/01、视图解析与渲染#视图解析机制]]
