## 一、SpringMVC 基础概念

###### 1. 什么是 SpringMVC?
###### 2. SpringMVC 的核心组件有哪些?
###### 3. SpringMVC 的优点有哪些?
###### 4. SpringMVC 的工作原理是什么?

## 二、DispatcherServlet 与请求处理流程

###### 1. 什么是 DispatcherServlet?
###### 2. DispatcherServlet 的作用是什么?
###### 3. SpringMVC 的请求处理流程是怎样的?
###### 4. 前端控制器的作用是什么?
###### 5. HandlerMapping 的作用是什么?
###### 6. HandlerAdapter 的作用是什么?
###### 7. ViewResolver 的作用是什么?

## 三、Controller 与请求映射

###### 1. @Controller 注解的作用是什么?
###### 2. @RequestMapping 注解的作用是什么?
###### 3. @GetMapping 和 @PostMapping 的区别是什么?
###### 4. @PathVariable 注解的作用是什么?
###### 5. @RequestParam 注解的作用是什么?
###### 6. @RequestBody 注解的作用是什么?
###### 7. @ResponseBody 注解的作用是什么?
###### 8. @RestController 和 @Controller 的区别是什么?
###### 9. 如何在 SpringMVC 中处理 RESTful 请求?
###### 10. @RequestHeader 注解的作用是什么?
###### 11. @CookieValue 注解的作用是什么?
###### 12. @ModelAttribute 注解的作用是什么?
###### 13. @SessionAttributes 注解的作用是什么?

## 四、参数绑定与数据转换

###### 1. SpringMVC 如何进行参数绑定?
###### 2. SpringMVC 支持哪些参数类型?
###### 3. 如何在 SpringMVC 中接收 JSON 数据?
###### 4. 如何在 SpringMVC 中接收 XML 数据?
###### 5. 什么是数据绑定?
###### 6. 如何自定义类型转换器?
###### 7. WebDataBinder 的作用是什么?
###### 8. @InitBinder 注解的作用是什么?
###### 9. 如何处理日期类型的参数?
###### 10. PropertyEditor 和 Converter 的区别是什么?

## 五、数据校验

###### 1. SpringMVC 如何进行数据校验?
###### 2. @Valid 和 @Validated 的区别是什么?
###### 3. BindingResult 的作用是什么?
###### 4. 如何自定义校验规则?
###### 5. JSR-303 规范是什么?
###### 6. 常用的校验注解有哪些?
###### 7. 如何实现分组校验?

## 六、视图解析与渲染

###### 1. SpringMVC 支持哪些视图技术?
###### 2. InternalResourceViewResolver 的作用是什么?
###### 3. 如何配置视图解析器?
###### 4. 什么是视图名称解析?
###### 5. 如何返回 JSON 数据?
###### 6. 如何返回 XML 数据?
###### 7. ModelAndView 的作用是什么?
###### 8. Model、ModelMap 和 Map 的区别是什么?
###### 9. RedirectAttributes 的作用是什么?
###### 10. 如何实现页面重定向?
###### 11. 如何实现请求转发?
###### 12. redirect 和 forward 的区别是什么?

## 七、拦截器

###### 1. 什么是拦截器?
###### 2. 拦截器和过滤器的区别是什么?
###### 3. 如何自定义拦截器?
###### 4. HandlerInterceptor 接口有哪些方法?
###### 5. preHandle、postHandle 和 afterCompletion 方法的执行顺序是什么?
###### 6. 多个拦截器的执行顺序是怎样的?
###### 7. 如何配置拦截器?
###### 8. 拦截器的应用场景有哪些?

## 八、异常处理

###### 1. SpringMVC 如何处理异常?
###### 2. @ExceptionHandler 注解的作用是什么?
###### 3. @ControllerAdvice 注解的作用是什么?
###### 4. @RestControllerAdvice 和 @ControllerAdvice 的区别是什么?
###### 5. SimpleMappingExceptionResolver 的作用是什么?
###### 6. 如何实现全局异常处理?
###### 7. 如何自定义异常类?
###### 8. HandlerExceptionResolver 的作用是什么?

## 九、文件上传与下载

###### 1. SpringMVC 如何实现文件上传?
###### 2. MultipartFile 的作用是什么?
###### 3. 如何配置文件上传解析器?
###### 4. 如何限制上传文件的大小?
###### 5. 如何实现多文件上传?
###### 6. SpringMVC 如何实现文件下载?
###### 7. ResponseEntity 的作用是什么?

## 十、RESTful 风格

###### 1. 什么是 RESTful?
###### 2. RESTful 的设计原则有哪些?
###### 3. SpringMVC 如何支持 RESTful?
###### 4. @PutMapping 和 @DeleteMapping 的作用是什么?
###### 5. @PatchMapping 的作用是什么?
###### 6. 如何处理 RESTful 风格的 URL?
###### 7. HiddenHttpMethodFilter 的作用是什么?

## 十一、静态资源处理

###### 1. SpringMVC 如何处理静态资源?
###### 2. mvc:resources 标签的作用是什么?
###### 3. mvc:default-servlet-handler 的作用是什么?
###### 4. ResourceHandlerRegistry 的作用是什么?

## 十二、跨域请求处理

###### 1. 什么是跨域问题?
###### 2. SpringMVC 如何解决跨域问题?
###### 3. @CrossOrigin 注解的作用是什么?
###### 4. 如何全局配置跨域?
###### 5. CorsRegistry 的作用是什么?

## 十三、配置与注解驱动

###### 1. mvc:annotation-driven 的作用是什么?
###### 2. @EnableWebMvc 注解的作用是什么?
###### 3. WebMvcConfigurer 接口的作用是什么?
###### 4. 如何自定义 SpringMVC 配置?
###### 5. 如何配置消息转换器?
###### 6. HttpMessageConverter 的作用是什么?
###### 7. 如何配置内容协商?

## 十四、Session 与 Cookie

###### 1. SpringMVC 如何操作 Session?
###### 2. SpringMVC 如何操作 Cookie?
###### 3. @SessionAttribute 注解的作用是什么?
###### 4. HttpSession 和 @SessionAttributes 的区别是什么?

## 十五、异步请求处理

###### 1. SpringMVC 如何支持异步请求?
###### 2. Callable 和 DeferredResult 的作用是什么?
###### 3. @Async 注解在 SpringMVC 中的应用是什么?
###### 4. 异步请求处理的原理是什么?

## 十六、国际化

###### 1. SpringMVC 如何实现国际化?
###### 2. LocaleResolver 的作用是什么?
###### 3. 如何配置国际化资源文件?
###### 4. MessageSource 的作用是什么?

## 十七、整合其他技术

###### 1. SpringMVC 如何整合 MyBatis?
###### 2. SpringMVC 如何整合 Hibernate?
###### 3. SpringMVC 如何整合 Spring Security?
###### 4. SpringMVC 如何整合 Thymeleaf?
###### 5. SpringMVC 如何整合 FreeMarker?
###### 6. SpringMVC 如何整合 WebSocket?

## 十八、性能优化与最佳实践

###### 1. SpringMVC 的性能优化方法有哪些?
###### 2. 如何避免重复提交?
###### 3. 如何实现请求幂等性?
###### 4. SpringMVC 的最佳实践有哪些?

## 十九、源码与原理

###### 1. DispatcherServlet 的源码分析
###### 2. HandlerMapping 的实现原理是什么?
###### 3. HandlerAdapter 的实现原理是什么?
###### 4. 参数解析器的工作原理是什么?
###### 5. 返回值处理器的工作原理是什么?
###### 6. SpringMVC 的九大组件是什么?