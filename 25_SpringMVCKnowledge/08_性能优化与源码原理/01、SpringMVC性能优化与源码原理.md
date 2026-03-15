# SpringMVC 性能优化与源码原理

## 性能优化策略

**1. 启用异步处理**：对于耗时操作，使用 `Callable` 或 `DeferredResult` 释放 Tomcat 请求线程，配合自定义线程池：

```java
@Configuration
@EnableAsync
public class AsyncConfig implements WebMvcConfigurer {
    @Override
    public void configureAsyncSupport(AsyncSupportConfigurer configurer) {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(10);
        executor.setMaxPoolSize(50);
        executor.setQueueCapacity(200);
        executor.setThreadNamePrefix("mvc-async-");
        executor.initialize();
        configurer.setTaskExecutor(executor);
        configurer.setDefaultTimeout(30000);  // 30 秒超时
    }
}
```

**2. 静态资源缓存**：配置 `Cache-Control` 头，让浏览器缓存静态资源，减少重复请求：
```java
registry.addResourceHandler("/static/**")
        .setCacheControl(CacheControl.maxAge(365, TimeUnit.DAYS).cachePublic());
```

**3. 视图缓存**：ViewResolver 缓存解析结果，避免重复解析：
```java
resolver.setCache(true);
resolver.setCacheLimit(1024);
```

**4. 精准路径匹配**：减少 HandlerMapping 的匹配范围，路径越精确，匹配越快。

**5. 控制器方法优化**：避免在 Controller 中执行耗时操作，业务逻辑下沉到 Service 层；利用 Spring 缓存（`@Cacheable`）减少重复计算。

---

## 防止重复提交

**Token 机制（推荐）**：
1. 用户获取表单时，服务端生成唯一 Token 存入 Redis
2. 表单提交时携带 Token
3. 服务端验证 Token 存在，随即删除（原子操作保证只处理一次）
4. Token 不存在则拒绝请求

```java
@Aspect
@Component
public class PreventDuplicateAspect {
    @Autowired
    private StringRedisTemplate redisTemplate;
    
    @Around("@annotation(preventDuplicate)")
    public Object around(ProceedingJoinPoint pjp, PreventDuplicate preventDuplicate) throws Throwable {
        String key = "prevent:" + getKey(pjp, preventDuplicate);
        Boolean success = redisTemplate.opsForValue()
            .setIfAbsent(key, "1", Duration.ofSeconds(preventDuplicate.expire()));
        if (!Boolean.TRUE.equals(success)) {
            throw new BusinessException(429, "请勿重复提交");
        }
        return pjp.proceed();
    }
}
```

---

## 接口幂等性

幂等性要求：对于同一个操作，执行一次和执行多次的效果相同。

实现层次：

**HTTP 层面**：GET/PUT/DELETE 天然幂等，POST 需要额外控制。

**业务层面**：在执行操作前检查状态，比如转账前检查是否已经转过（通过订单号唯一约束）。

**数据库层面**：唯一索引 + 乐观锁，防止并发重复插入：

```java
@Transactional
public Order createOrder(CreateOrderDTO dto) {
    // 唯一约束：同一用户同一商品同一时间窗口只能有一条待支付订单
    if (orderRepository.existsByUserIdAndProductIdAndStatus(dto.getUserId(), dto.getProductId(), "PENDING")) {
        throw new BusinessException(409, "订单已存在");
    }
    return orderRepository.save(buildOrder(dto));
}
```

---

## HandlerMapping 源码原理

`RequestMappingHandlerMapping` 的工作分两个阶段：

**初始化阶段**：`ApplicationContext` 刷新后，调用 `detectHandlerMethods()`，扫描所有 `@Controller` Bean，找到所有 `@RequestMapping` 注解的方法，封装为 `HandlerMethod`（包含 Bean 实例 + 方法反射对象），注册到 `MappingRegistry` 中。

**请求匹配阶段**：`lookupHandlerMethod()` 根据请求 URL 查找匹配的 `HandlerMethod`，支持精确匹配和模式匹配（含路径变量的 URL 需要模式匹配，`PathPattern` 或 Ant 风格），多个匹配时按精确度排序，取最佳匹配。

---

## HandlerAdapter 源码原理

`RequestMappingHandlerAdapter` 核心执行流程（在 `invokeHandlerMethod()` 中）：

1. **参数解析**：遍历方法参数，找到对应的 `HandlerMethodArgumentResolver` 解析参数值（`@PathVariable`→`PathVariableMethodArgumentResolver`、`@RequestBody`→`RequestResponseBodyMethodProcessor`）
2. **方法执行**：通过 `InvocableHandlerMethod.invokeForRequest()` 反射调用目标方法
3. **返回值处理**：找到对应的 `HandlerMethodReturnValueHandler` 处理返回值（有 `@ResponseBody` → 写响应体；String → 解析为视图名；`ModelAndView` → 提取模型和视图）

---

## 参数解析器工作原理

`HandlerMethodArgumentResolver` 接口：
- `supportsParameter()`：判断是否能处理该参数
- `resolveArgument()`：解析参数值

参数解析器以列表形式注册，遍历列表找到第一个 `supportsParameter()` 返回 true 的解析器来处理。

常见解析器对应关系：
- `@RequestParam` → `RequestParamMethodArgumentResolver`
- `@PathVariable` → `PathVariableMethodArgumentResolver`
- `@RequestBody` → `RequestResponseBodyMethodProcessor`
- `@ModelAttribute` → `ModelAttributeMethodProcessor`
- `Model`/`ModelMap` → `ModelMethodProcessor`
- `HttpServletRequest` → `ServletRequestMethodArgumentResolver`

---

## 返回值处理器工作原理

`HandlerMethodReturnValueHandler` 接口与参数解析器同理，也是遍历列表匹配处理器。

`RequestResponseBodyMethodProcessor` 处理 `@ResponseBody` 的流程：
1. 标记请求已处理（不再走视图解析）
2. 内容协商：根据请求 `Accept` 头和可用的 `HttpMessageConverter` 确定响应格式
3. 调用匹配的 `HttpMessageConverter.write()` 将返回值序列化到响应输出流

---

## 相关面试题 →

[[../../10_Developlanguage/005_Spring/02_SpringMVCSubject/18、性能优化与最佳实践]]
[[../../10_Developlanguage/005_Spring/02_SpringMVCSubject/19、源码与原理]]
