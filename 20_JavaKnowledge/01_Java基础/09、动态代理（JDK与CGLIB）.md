# 动态代理（JDK & CGLIB）

> **核心关键词**：代理模式、JDK动态代理、CGLIB字节码增强、InvocationHandler、Spring AOP

---

## 一、代理模式概述

代理模式为目标对象提供一个代理对象，通过代理对象来控制对目标对象的访问。

```
客户端 → 代理对象 → 目标对象
             ↓
        (增强逻辑：日志/事务/权限/缓存)
```

**动态代理的核心价值**：不需要为每个目标类手写代理类，在**运行时动态生成**代理类。这是 Spring AOP、MyBatis MapperProxy、RPC 框架的基础。

---

## 二、JDK 动态代理

### 2.1 基本原理

JDK 动态代理基于**接口**，要求目标类必须实现接口。运行时通过反射在内存中动态生成代理类的字节码。

**核心 API**：
- `java.lang.reflect.Proxy`：生成代理类的工厂
- `java.lang.reflect.InvocationHandler`：代理类的增强逻辑

### 2.2 使用示例

```java
// 1. 定义接口
public interface UserService {
    User findById(Long id);
    void save(User user);
}

// 2. 真实实现
public class UserServiceImpl implements UserService {
    @Override
    public User findById(Long id) {
        System.out.println("查询用户: " + id);
        return new User(id, "张三");
    }
    @Override
    public void save(User user) {
        System.out.println("保存用户: " + user);
    }
}

// 3. InvocationHandler（增强逻辑）
public class LoggingHandler implements InvocationHandler {
    private final Object target;  // 目标对象
    
    public LoggingHandler(Object target) {
        this.target = target;
    }
    
    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        long start = System.currentTimeMillis();
        System.out.println("[LOG] 开始执行: " + method.getName());
        
        try {
            Object result = method.invoke(target, args);  // 调用目标方法
            System.out.println("[LOG] 执行成功，耗时: " + (System.currentTimeMillis() - start) + "ms");
            return result;
        } catch (InvocationTargetException e) {
            System.out.println("[LOG] 执行失败: " + e.getCause().getMessage());
            throw e.getCause();  // 解包 InvocationTargetException
        }
    }
}

// 4. 创建代理对象
UserService target = new UserServiceImpl();
UserService proxy = (UserService) Proxy.newProxyInstance(
    target.getClass().getClassLoader(),     // 类加载器
    target.getClass().getInterfaces(),      // 代理的接口列表
    new LoggingHandler(target)              // 增强逻辑
);

// 5. 使用（调用 proxy 的任何方法都会走 LoggingHandler.invoke）
User user = proxy.findById(1L);
```

### 2.3 JDK 代理的工作原理

```java
// Proxy.newProxyInstance 在运行时生成的代理类（伪代码）
public final class $Proxy0 extends Proxy implements UserService {
    
    private static Method m1; // findById 方法引用
    private static Method m2; // save 方法引用
    
    static {
        m1 = UserService.class.getMethod("findById", Long.class);
        m2 = UserService.class.getMethod("save", User.class);
    }
    
    $Proxy0(InvocationHandler h) { super(h); }
    
    @Override
    public User findById(Long id) {
        return (User) h.invoke(this, m1, new Object[]{id});  // 委托给 InvocationHandler
    }
    
    @Override
    public void save(User user) {
        h.invoke(this, m2, new Object[]{user});
    }
}
```

---

## 三、CGLIB 动态代理

### 3.1 基本原理

CGLIB（Code Generation Library）通过 **ASM 字节码框架**在运行时生成目标类的**子类**来实现代理，不需要接口。

```
目标类 UserServiceImpl
        ↓ CGLIB 生成子类
代理类 UserServiceImpl$$EnhancerByCGLIB$$xxxxx
   继承 UserServiceImpl，重写所有非 final 方法，在方法前后插入增强逻辑
```

**限制**：
- `final` 类不能被代理（无法继承）
- `final` 方法不能被增强（无法重写）

### 3.2 使用示例

```java
// 不需要接口！直接代理普通类
public class OrderService {
    public Order createOrder(String productId) {
        System.out.println("创建订单: " + productId);
        return new Order(productId);
    }
}

// Maven 依赖
// <dependency><groupId>cglib</groupId><artifactId>cglib</artifactId></dependency>

// CGLIB 代理
Enhancer enhancer = new Enhancer();
enhancer.setSuperclass(OrderService.class);  // 设置父类
enhancer.setCallback(new MethodInterceptor() {
    @Override
    public Object intercept(Object obj, Method method, Object[] args, 
                            MethodProxy proxy) throws Throwable {
        System.out.println("[CGLIB] 方法前: " + method.getName());
        Object result = proxy.invokeSuper(obj, args);  // 调用父类（目标）方法
        System.out.println("[CGLIB] 方法后: " + method.getName());
        return result;
    }
});

OrderService proxy = (OrderService) enhancer.create();
proxy.createOrder("PROD-001");
```

---

## 四、JDK 代理 vs CGLIB 对比

| 对比项 | JDK 动态代理 | CGLIB |
|--------|------------|-------|
| 代理方式 | 基于**接口**，生成实现类 | 基于**继承**，生成子类 |
| 是否需要接口 | **必须**实现接口 | 不需要接口 |
| final 限制 | 接口方法不能是 final | **final 类和 final 方法不能代理** |
| 性能（创建）| 快（JDK 内置）| 慢（字节码生成开销）|
| 性能（调用）| JDK 8 以后差距很小 | 通过 FastClass 索引，调用快 |
| 依赖 | JDK 内置，无需额外依赖 | 需要引入 cglib 依赖 |
| Spring 中的选择 | 目标类有接口时使用 | 目标类没有接口时，或配置了 proxyTargetClass=true |

---

## 五、Spring AOP 中的动态代理

### 5.1 Spring 的选择策略

```java
// Spring Boot 2.0+ 默认使用 CGLIB（无论是否有接口）
// 可以通过配置改变：
@SpringBootApplication
public class App {
    public static void main(String[] args) {
        SpringApplication.run(App.class, args);
    }
}

// application.properties
spring.aop.proxy-target-class=true  // true=强制CGLIB，false=有接口用JDK

// 注解方式强制使用 CGLIB
@EnableAspectJAutoProxy(proxyTargetClass = true)
```

### 5.2 Spring AOP 的工作流程

```java
// 1. 定义切面
@Aspect
@Component
public class LogAspect {
    
    @Before("@annotation(Log)")
    public void before(JoinPoint jp) {
        System.out.println("方法前: " + jp.getSignature().getName());
    }
    
    @AfterReturning(pointcut = "@annotation(Log)", returning = "result")
    public void afterReturning(JoinPoint jp, Object result) {
        System.out.println("方法后, 返回值: " + result);
    }
    
    @Around("execution(* com.example.service.*.*(..))")
    public Object around(ProceedingJoinPoint pjp) throws Throwable {
        long start = System.currentTimeMillis();
        try {
            Object result = pjp.proceed();  // 调用目标方法
            return result;
        } finally {
            System.out.println("耗时: " + (System.currentTimeMillis() - start) + "ms");
        }
    }
}

// 2. 标记需要增强的方法
@Service
public class UserService {
    
    @Log  // 自定义注解，触发切面
    public User findById(Long id) {
        return userRepository.findById(id);
    }
}
```

### 5.3 Spring 代理失效的场景

```java
@Service
public class OrderService {
    
    @Transactional
    public void createOrder() {
        // ...
        this.updateStock();  // ❌ this 调用不走代理，事务注解失效！
    }
    
    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void updateStock() {
        // ...
    }
}

// ✅ 解决方案1：注入自身代理对象
@Service
public class OrderService {
    @Autowired
    private OrderService self;  // 注入自己（Spring 注入的是代理对象）
    
    public void createOrder() {
        self.updateStock();  // ✅ 通过代理对象调用，事务生效
    }
}

// ✅ 解决方案2：通过 ApplicationContext 获取代理
OrderService proxy = applicationContext.getBean(OrderService.class);
proxy.updateStock();

// ✅ 解决方案3：开启 @EnableAspectJAutoProxy(exposeProxy = true)
OrderService proxy = (OrderService) AopContext.currentProxy();
proxy.updateStock();
```

---

## 六、面试要点速查

| 问题 | 要点 |
|------|------|
| JDK 代理和 CGLIB 的区别 | JDK 基于接口生成实现类；CGLIB 通过继承生成子类，不需要接口 |
| 为什么 JDK 代理需要接口 | 代理类继承 Proxy 类（Java 单继承），只能通过接口来规范代理对象的行为 |
| CGLIB 不能代理 final 的原因 | final 方法/类不能被继承和重写 |
| Spring 什么时候用 JDK，什么时候用 CGLIB | Spring Boot 2.0+ 默认用 CGLIB；也可通过 proxyTargetClass 配置 |
| 代理方法内部调用自身方法为什么失效 | `this` 是目标对象本身，不是代理对象，调用不经过 AOP 拦截 |
| InvocationHandler 中为何需要解包 InvocationTargetException | `method.invoke()` 将目标方法抛出的异常包装成 InvocationTargetException，需要取 getCause() |


---

**相关面试题** → [[../../10_Developlanguage/001_java/01_JavaBaseSubject/14、代理|14、代理]]
