# DispatcherServlet 请求处理源码解析

> **核心类**：`DispatcherServlet`  
> **核心包**：`org.springframework.web.servlet`  
> **继承链**：`HttpServlet` → `HttpServletBean` → `FrameworkServlet` → `DispatcherServlet`

---

## 一、Spring MVC 整体架构

```
HTTP 请求
└── DispatcherServlet.doDispatch()
    ├── HandlerMapping.getHandler()           ← ① 找处理器（Controller 方法）
    │   └── HandlerExecutionChain（handler + interceptors）
    ├── HandlerAdapter.handle()               ← ② 执行处理器
    │   ├── HandlerInterceptor.preHandle()    ← 拦截器前置
    │   ├── 执行 Controller 方法
    │   │   ├── HandlerMethodArgumentResolver  ← 参数解析（@RequestBody等）
    │   │   └── HandlerMethodReturnValueHandler ← 返回值处理（@ResponseBody等）
    │   └── HandlerInterceptor.postHandle()   ← 拦截器后置
    ├── ExceptionHandlerExceptionResolver     ← ③ 异常处理（@ExceptionHandler）
    └── ViewResolver.resolveViewName()        ← ④ 视图解析（返回非 @ResponseBody 时）
```

---

## 二、DispatcherServlet 初始化流程

```java
// DispatcherServlet.java
@Override
protected void onRefresh(ApplicationContext context) {
    initStrategies(context);
}

protected void initStrategies(ApplicationContext context) {
    initMultipartResolver(context);           // 文件上传解析器
    initLocaleResolver(context);              // 国际化解析器
    initThemeResolver(context);               // 主题解析器
    initHandlerMappings(context);             // ★ 初始化 HandlerMapping
    initHandlerAdapters(context);             // ★ 初始化 HandlerAdapter
    initHandlerExceptionResolvers(context);   // ★ 异常处理器
    initRequestToViewNameTranslator(context); // 请求到视图名转换器
    initViewResolvers(context);               // ★ 视图解析器
    initFlashMapManager(context);             // FlashMap 管理器
}

// initHandlerMappings 示例
private void initHandlerMappings(ApplicationContext context) {
    this.handlerMappings = null;
    if (this.detectAllHandlerMappings) {
        // 从容器中获取所有 HandlerMapping Bean
        Map<String, HandlerMapping> matchingBeans = BeanFactoryUtils
                .beansOfTypeIncludingAncestors(context, HandlerMapping.class, true, false);
        if (!matchingBeans.isEmpty()) {
            this.handlerMappings = new ArrayList<>(matchingBeans.values());
            // 按 @Order 排序
            AnnotationAwareOrderComparator.sort(this.handlerMappings);
        }
    }
    // 如果没有找到，使用默认策略（DispatcherServlet.properties 中配置）
    if (this.handlerMappings == null) {
        this.handlerMappings = getDefaultStrategies(context, HandlerMapping.class);
    }
}
```

---

## 三、doDispatch() —— 请求处理核心

```java
// DispatcherServlet.java
protected void doDispatch(HttpServletRequest request, HttpServletResponse response) throws Exception {
    HttpServletRequest processedRequest = request;
    HandlerExecutionChain mappedHandler = null;
    boolean multipartRequestParsed = false;
    WebAsyncManager asyncManager = WebAsyncUtils.getAsyncManager(request);

    try {
        ModelAndView mv = null;
        Exception dispatchException = null;

        try {
            // ① 文件上传请求预处理
            processedRequest = checkMultipart(request);
            multipartRequestParsed = (processedRequest != request);

            // ② ★ 查找 Handler（Controller 方法）
            mappedHandler = getHandler(processedRequest);
            if (mappedHandler == null) {
                noHandlerFound(processedRequest, response);  // 404
                return;
            }

            // ③ 找到对应的 HandlerAdapter（支持执行该 Handler 的适配器）
            HandlerAdapter ha = getHandlerAdapter(mappedHandler.getHandler());

            // ④ 处理 Last-Modified 缓存头（GET/HEAD 请求）
            String method = request.getMethod();
            boolean isGet = HttpMethod.GET.matches(method);
            if (isGet || HttpMethod.HEAD.matches(method)) {
                long lastModified = ha.getLastModified(request, mappedHandler.getHandler());
                if (new ServletWebRequest(request, response).checkNotModified(lastModified) && isGet) {
                    return;  // 304 Not Modified
                }
            }

            // ⑤ ★ 执行拦截器 preHandle（返回 false 则终止）
            if (!mappedHandler.applyPreHandle(processedRequest, response)) {
                return;
            }

            // ⑥ ★ 执行 Handler（Controller 方法）
            mv = ha.handle(processedRequest, response, mappedHandler.getHandler());

            // 异步请求直接返回
            if (asyncManager.isConcurrentHandlingStarted()) {
                return;
            }

            // ⑦ 如果返回的 ModelAndView 没有视图名，使用默认视图名
            applyDefaultViewName(processedRequest, mv);

            // ⑧ ★ 执行拦截器 postHandle
            mappedHandler.applyPostHandle(processedRequest, response, mv);
        } catch (Exception ex) {
            dispatchException = ex;
        } catch (Throwable err) {
            dispatchException = new NestedServletException("Handler dispatch failed", err);
        }

        // ⑨ 处理结果（渲染视图 或 处理异常）
        processDispatchResult(processedRequest, response, mappedHandler, mv, dispatchException);
    }
    catch (Exception ex) {
        triggerAfterCompletion(processedRequest, response, mappedHandler, ex);
        throw ex;
    }
    finally {
        if (multipartRequestParsed) {
            cleanupMultipart(processedRequest);
        }
    }
}
```

---

## 四、getHandler() —— HandlerMapping 查找流程

```java
// DispatcherServlet.java
@Nullable
protected HandlerExecutionChain getHandler(HttpServletRequest request) throws Exception {
    if (this.handlerMappings != null) {
        // 遍历所有 HandlerMapping，找到能处理该请求的
        for (HandlerMapping mapping : this.handlerMappings) {
            HandlerExecutionChain handler = mapping.getHandler(request);
            if (handler != null) {
                return handler;
            }
        }
    }
    return null;
}

// RequestMappingHandlerMapping.getHandler() → 父类 AbstractHandlerMapping.getHandler()
@Override
@Nullable
public final HandlerExecutionChain getHandler(HttpServletRequest request) throws Exception {
    // ★ 按 URL + HTTP 方法查找 Handler
    Object handler = getHandlerInternal(request);
    if (handler == null) {
        handler = getDefaultHandler();
    }
    if (handler == null) {
        return null;
    }
    if (handler instanceof String handlerName) {
        handler = obtainApplicationContext().getBean(handlerName);
    }

    // ★ 将 Handler + 所有匹配的 Interceptor 打包成 HandlerExecutionChain
    HandlerExecutionChain executionChain = getHandlerExecutionChain(handler, request);

    // 处理 CORS
    if (hasCorsConfigurationSource(handler) || CorsUtils.isPreFlightRequest(request)) {
        CorsConfiguration config = getCorsConfiguration(handler, request);
        executionChain = getCorsHandlerExecutionChain(request, executionChain, config);
    }

    return executionChain;
}
```

---

## 五、RequestMappingHandlerMapping —— @RequestMapping 注册与查找

```java
// RequestMappingHandlerMapping.java
// 继承了 AbstractHandlerMethodMapping，在容器初始化时扫描所有 @RequestMapping

@Override
protected boolean isHandler(Class<?> beanType) {
    // 有 @Controller 或 @RequestMapping 注解的类才是 Handler
    return (AnnotatedElementUtils.hasAnnotation(beanType, Controller.class) ||
            AnnotatedElementUtils.hasAnnotation(beanType, RequestMapping.class));
}

@Override
@Nullable
protected RequestMappingInfo getMappingForMethod(Method method, Class<?> handlerType) {
    // 从方法上获取 @RequestMapping 信息
    RequestMappingInfo info = createRequestMappingInfo(method);
    if (info != null) {
        // 合并类级别的 @RequestMapping（URL 前缀）
        RequestMappingInfo typeInfo = createRequestMappingInfo(handlerType);
        if (typeInfo != null) {
            info = typeInfo.combine(info);
        }
    }
    return info;
}

// ★ 注册阶段：afterPropertiesSet → detectHandlerMethods → 遍历所有 Bean 注册
@Override
protected void detectHandlerMethods(Object handler) {
    Class<?> handlerType = (handler instanceof String beanName
            ? obtainApplicationContext().getType(beanName) : handler.getClass());
    if (handlerType != null) {
        Class<?> userType = ClassUtils.getUserClass(handlerType);
        // 获取所有 @RequestMapping 方法及对应的映射信息
        Map<Method, RequestMappingInfo> methods = MethodIntrospector.selectMethods(userType,
                (MethodIntrospector.MetadataLookup<RequestMappingInfo>) method -> {
                    return getMappingForMethod(method, userType);
                });
        methods.forEach((method, mapping) -> {
            Method invocableMethod = AopUtils.selectInvocableMethod(method, userType);
            // 注册到 MappingRegistry
            registerHandlerMethod(handler, invocableMethod, mapping);
        });
    }
}
```

---

## 六、RequestMappingHandlerAdapter —— 执行 Controller 方法

```java
// RequestMappingHandlerAdapter.java
@Override
protected ModelAndView handleInternal(HttpServletRequest request,
        HttpServletResponse response, HandlerMethod handlerMethod) throws Exception {
    ModelAndView mav;
    checkRequest(request);

    // 是否需要同步 Session
    if (this.synchronizeOnSession) {
        HttpSession session = request.getSession(false);
        if (session != null) {
            Object mutex = WebUtils.getSessionMutex(session);
            synchronized (mutex) {
                mav = invokeHandlerMethod(request, response, handlerMethod);
            }
        } else {
            mav = invokeHandlerMethod(request, response, handlerMethod);
        }
    } else {
        mav = invokeHandlerMethod(request, response, handlerMethod);
    }

    // 处理缓存控制头
    if (!response.containsHeader(HEADER_CACHE_CONTROL)) {
        if (getSessionAttributesHandler(handlerMethod).hasSessionAttributes()) {
            applyCacheSeconds(response, this.cacheSecondsForSessionAttributeHandlers);
        } else {
            prepareResponse(response);
        }
    }
    return mav;
}

@Nullable
protected ModelAndView invokeHandlerMethod(HttpServletRequest request,
        HttpServletResponse response, HandlerMethod handlerMethod) throws Exception {

    ServletWebRequest webRequest = new ServletWebRequest(request, response);
    try {
        WebDataBinderFactory binderFactory = getDataBinderFactory(handlerMethod);
        ModelFactory modelFactory = getModelFactory(handlerMethod, binderFactory);

        // ★ 创建 ServletInvocableHandlerMethod（可调用的 Handler 方法）
        ServletInvocableHandlerMethod invocableMethod = createInvocableHandlerMethod(handlerMethod);
        if (this.argumentResolvers != null) {
            invocableMethod.setHandlerMethodArgumentResolvers(this.argumentResolvers);
        }
        if (this.returnValueHandlers != null) {
            invocableMethod.setHandlerMethodReturnValueHandlers(this.returnValueHandlers);
        }
        invocableMethod.setDataBinderFactory(binderFactory);
        invocableMethod.setParameterNameDiscoverer(this.parameterNameDiscoverer);

        ModelAndViewContainer mavContainer = new ModelAndViewContainer();
        mavContainer.addAllAttributes(RequestContextUtils.getInputFlashMap(request));
        modelFactory.initModel(webRequest, mavContainer, invocableMethod);
        mavContainer.setIgnoreDefaultModelOnRedirect(this.ignoreDefaultModelOnRedirect);

        // ★ 调用 Controller 方法（参数解析 + 方法反射调用 + 返回值处理）
        invocableMethod.invokeAndHandle(webRequest, mavContainer);

        if (asyncManager.isConcurrentHandlingStarted()) {
            return null;
        }

        return getModelAndView(mavContainer, modelFactory, webRequest);
    } finally {
        webRequest.requestCompleted();
    }
}
```

---

## 七、参数解析 —— HandlerMethodArgumentResolver

```java
// InvocableHandlerMethod.java
protected Object[] getMethodArgumentValues(NativeWebRequest request,
        @Nullable ModelAndViewContainer mavContainer, Object... providedArgs) throws Exception {

    MethodParameter[] parameters = getMethodParameters();
    Object[] args = new Object[parameters.length];

    for (int i = 0; i < parameters.length; i++) {
        MethodParameter parameter = parameters[i];
        parameter.initParameterNameDiscovery(this.parameterNameDiscoverer);
        args[i] = findProvidedArgument(parameter, providedArgs);
        if (args[i] != null) continue;

        if (!this.resolvers.supportsParameter(parameter)) {
            throw new IllegalStateException(...);
        }
        // ★ 找到支持该参数的 Resolver 并解析
        args[i] = this.resolvers.resolveArgument(parameter, mavContainer, request, this.dataBinderFactory);
    }
    return args;
}

// 常用参数解析器：
// RequestParamMethodArgumentResolver     ← @RequestParam
// PathVariableMethodArgumentResolver     ← @PathVariable
// RequestBodyMethodProcessor             ← @RequestBody（含 JSON 反序列化）
// ModelAttributeMethodProcessor          ← @ModelAttribute / 对象参数
// SessionAttributeMethodArgumentResolver ← @SessionAttribute
// HeaderMethodArgumentResolver           ← @RequestHeader
```

---

## 八、@ResponseBody 处理 —— HttpMessageConverter

```java
// RequestResponseBodyMethodProcessor.java（同时处理 @RequestBody 和 @ResponseBody）
@Override
public void handleReturnValue(@Nullable Object returnValue,
        MethodParameter returnType, ModelAndViewContainer mavContainer,
        NativeWebRequest webRequest) throws IOException, HttpMediaTypeNotAcceptableException,
        HttpMessageNotWritableException {

    mavContainer.setRequestHandled(true);
    ServletServerHttpRequest inputMessage = createInputMessage(webRequest);
    ServletServerHttpResponse outputMessage = createOutputMessage(webRequest);

    // ★ 根据 Accept 头和返回类型选择合适的 HttpMessageConverter 写出
    writeWithMessageConverters(returnValue, returnType, inputMessage, outputMessage);
}

// 常用 HttpMessageConverter：
// MappingJackson2HttpMessageConverter  ← JSON（Jackson）
// StringHttpMessageConverter           ← String 类型
// ByteArrayHttpMessageConverter        ← byte[]
// FormHttpMessageConverter             ← 表单数据
```

---

## 九、常见面试问题

| 问题 | 答案要点 |
|------|---------|
| DispatcherServlet 的初始化流程？ | 继承 HttpServletBean，onRefresh 调用 initStrategies 初始化 9 大组件 |
| @RequestMapping 是如何注册和查找的？ | RequestMappingHandlerMapping 在 afterPropertiesSet 扫描并注册 MappingRegistry |
| @RequestBody 如何反序列化？ | RequestBodyMethodProcessor + MappingJackson2HttpMessageConverter |
| 拦截器（Interceptor）和过滤器（Filter）的区别？ | Filter 在 Servlet 容器级别；Interceptor 在 DispatcherServlet 内，可访问 Handler 信息 |
| @ExceptionHandler 如何生效？ | ExceptionHandlerExceptionResolver 在 processDispatchResult 中处理异常 |

---

## 十、相关源码文件

- [[../01_IoC容器源码/02、ApplicationContext启动流程（refresh方法）]]
- [[02、HandlerMapping与HandlerAdapter]]
- [[../06_SpringBoot自动配置源码/02、SpringMVC自动配置]]
