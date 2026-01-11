###### 1. 什么是 Spring Bean？
**Spring Bean**是由Spring IoC容器**实例化、组装和管理的对象**，是Spring应用程序的基本构建块。与普通Java对象不同，Bean的生命周期完全由Spring容器控制。
**核心特征**：
- **容器管理**：Bean由Spring IoC容器创建和管理，不直接通过`new`关键字实例化
- **配置元数据**：通过XML、注解或Java配置类定义Bean的创建规则
- **依赖注入**：容器自动处理Bean之间的依赖关系
- **生命周期回调**：支持初始化/销毁回调方法
**源码角度**：Spring通过`BeanDefinition`接口描述Bean的元数据，包括类名、作用域、属性值等。容器启动时，`BeanDefinitionReader`读取配置并注册到`BeanDefinitionRegistry`，最后由`BeanFactory`根据这些定义创建Bean实例。
**与普通对象的区别**：
```java
// 普通对象：直接new创建，完全由程序控制
User user = new User();

// Spring Bean：由容器管理，通过ApplicationContext获取
ApplicationContext context = new ClassPathXmlApplicationContext("config.xml");
User user = context.getBean("user", User.class);
```
###### 2. Spring 中 Bean 的作用域有哪些？
Spring支持6种Bean作用域，控制Bean实例的创建方式和生命周期：

|作用域|描述|适用场景|配置方式|
|---|---|---|---|
|**singleton**​|默认作用域，整个容器中只有一个实例|无状态Bean，如Service、DAO|`@Scope("singleton")`|
|**prototype**​|每次请求都创建新实例|有状态Bean，需要隔离的场景|`@Scope("prototype")`|
|**request**​|每次HTTP请求创建新实例|Web应用中的请求相关数据|`@Scope("request")`|
|**session**​|每个HTTP会话期间一个实例|用户会话数据管理|`@Scope("session")`|
|**application**​|整个Web应用生命周期一个实例|全局应用级数据|`@Scope("application")`|
|**websocket**​|每个WebSocket会话一个实例|WebSocket通信场景|`@Scope("websocket")`|
**源码机制**：作用域通过`Scope`接口实现。`SingletonScope`使用ConcurrentHashMap缓存实例，而`PrototypeScope`每次调用`getBean()`时都通过反射创建新对象。
**线程安全考虑**：singleton作用域的Bean需要确保线程安全，而prototype作用域每个线程有独立实例，天然线程安全但内存开销大。
###### 3. Spring Bean 的生命周期是怎样的？
Spring Bean的生命周期包含以下完整阶段：
1. **实例化**：容器调用构造函数创建Bean实例
2. **属性填充**：通过setter或字段注入依赖关系
3. **Aware接口回调**：执行`BeanNameAware`、`BeanFactoryAware`等接口方法
4. **前置处理**：`BeanPostProcessor`的`postProcessBeforeInitialization()`方法
5. **初始化**：执行`@PostConstruct`、`InitializingBean.afterPropertiesSet()`和自定义init方法
6. **后置处理**：`BeanPostProcessor`的`postProcessAfterInitialization()`方法
7. **使用期**：Bean完全初始化，可供业务使用
8. **销毁前处理**：执行`@PreDestroy`方法
9. **销毁**：执行`DisposableBean.destroy()`和自定义destroy方法
**源码体现**：在`AbstractAutowireCapableBeanFactory.doCreateBean()`方法中完整实现了这一流程。关键的`initializeBean()`方法处理初始化回调，而`destroyBean()`处理销毁逻辑。
###### 4. 如何在 Spring 中配置 Bean？
Spring提供三种主要的Bean配置方式：
**XML配置**（传统方式）：
```xml
<bean id="userService" class="com.example.UserService">
    <property name="userDao" ref="userDao"/>
    <property name="timeout" value="5000"/>
</bean>
```
**注解配置**（现代首选）：
```java
@Component
public class UserService {
    @Autowired
    private UserDao userDao;
}
```
**Java配置类**（显式控制）：
```java
@Configuration
public class AppConfig {
    @Bean
    public UserService userService() {
        return new UserService(userDao());
    }
    
    @Bean 
    public UserDao userDao() {
        return new UserDaoImpl();
    }
}
```
**选择建议**：现代Spring项目推荐以注解配置为主，复杂依赖或第三方库集成使用Java配置，遗留系统可能仍需XML配置。
###### 5. 什么是 Bean 的装配？有哪些装配方式？
**Bean装配**是指**创建Bean之间协作关系的过程**，即依赖注入的实现。主要有两种装配方式：
**显式装配**：手动明确指定依赖关系
- **构造器注入**：通过构造函数注入依赖
```java
public class UserService {
    private final UserDao userDao;
    
    public UserService(UserDao userDao) {
        this.userDao = userDao;
    }
}
```
- **Setter注入**：通过setter方法注入
```java
public class UserService {
    private UserDao userDao;
    
    public void setUserDao(UserDao userDao) {
        this.userDao = userDao;
    }
}
```
**隐式装配（自动装配）**：容器根据规则自动建立依赖关系，包括byType、byName等方式。
###### 6. 什么是自动装配？自动装配有哪些方式？
**自动装配**是Spring容器**自动建立Bean之间依赖关系**的机制，减少显式配置。
**自动装配模式**：

|模式|说明|优点|缺点|
|---|---|---|---|
|**byType**​|按类型自动装配|类型安全，配置简单|同类型多个Bean时失败|
|**byName**​|按名称自动装配|明确指定目标Bean|需要属性名与Bean名一致|
|**constructor**​|构造器参数自动装配|保证依赖完全初始化|复杂构造器时配置困难|
|**no**​|默认，不自动装配|完全控制依赖关系|配置工作量大|
**配置方式**：
```java
@Component
public class UserService {
    @Autowired // byType自动装配
    private UserDao userDao;
    
    @Autowired
    public UserService(@Qualifier("specificDao") UserDao dao) {
        // constructor自动装配
    }
}
```
**最佳实践**：推荐使用`@Autowired`进行byType装配，配合`@Qualifier`解决歧义性问题。
###### 7. @Autowired 和 @Resource 的区别是什么？
|特性|@Autowired|@Resource|
|---|---|---|
|**来源**​|Spring框架注解|JSR-250标准注解|
|**默认行为**​|按类型装配|按名称装配|
|**名称指定**​|需配合@Qualifier|自带name属性|
|**required属性**​|支持|不支持|
|**适用场景**​|纯Spring环境|需要标准化的场景|
**使用示例**：
```java
@Component
public class ExampleService {
    @Autowired // 按类型匹配
    @Qualifier("mainDao") // 明确指定Bean名称
    private UserDao userDao;
    
    @Resource(name = "secondaryDao") // 按名称匹配
    private UserDao anotherDao;
}
```
###### 8. @Component、@Service、@Controller、@Repository 的区别是什么？
这四个注解都是**组件扫描的标记注解**，本质功能相同，但用于不同层次体现语义差异：

|注解|层级|特殊功能|使用场景|
|---|---|---|---|
|**@Component**​|通用组件|无|通用组件类|
|**@Service**​|业务层|无|业务逻辑实现类|
|**@Controller**​|表现层|MVC控制器识别|Web请求处理类|
|**@Repository**​|持久层|自动转换数据访问异常|数据访问层类|
**源码角度**：这些注解都被`@Component`元注解标记，组件扫描时统一处理。`@Repository`通过`PersistenceExceptionTranslationPostProcessor`将特定数据访问异常转换为Spring的统一异常体系。
###### 9. 如何解决循环依赖问题？
**循环依赖**指Bean之间**相互依赖形成闭环**的情况，如A依赖B，B又依赖A。
**Spring的解决方案**：使用**三级缓存**机制：
1. **singletonObjects**：缓存完整的Bean实例
2. **earlySingletonObjects**：缓存提前暴露的原始对象
3. **singletonFactories**：缓存ObjectFactory，用于创建代理对象
**解决条件**：
- 仅适用于singleton作用域的Bean
- 依赖通过setter或字段注入（构造器注入无法解决）
- 没有自定义BeanPostProcessor干预创建过程
**避免策略**：
- 使用`@Lazy`延迟加载一方依赖
- 提取公共逻辑到第三个Bean中
- 使用ApplicationContext.getBean()显式获取依赖
###### 10. Spring 如何处理线程安全问题？
Spring本身不保证Bean的线程安全，需要开发者根据作用域采取不同策略：
**singleton Bean**：需要开发者保证线程安全
```java
@Service
public class StatelessService { // 无状态服务，线程安全
    public String process(String input) {
        return input.toUpperCase();
    }
}

@Service
public class StatefulService {
    private final ThreadLocal<User> userHolder = new ThreadLocal<>();
    
    public void setUser(User user) {
        userHolder.set(user); // 使用ThreadLocal保持线程隔离
    }
}
```
**prototype Bean**：每次请求新实例，天然线程安全但需注意性能
**最佳实践**：尽量使用无状态设计，避免在singleton Bean中维护可变状态。必须维护状态时使用ThreadLocal或并发集合。
###### 11. 什么是懒加载（Lazy Initialization）？
**懒加载**指**延迟Bean的实例化时机**，直到第一次被请求时才创建，而非容器启动时立即创建。
**配置方式**：
```java
@Component
@Lazy // 类级别懒加载
public class HeavyService {
    // 这个Bean在第一次被注入时才会实例化
}

@Configuration
public class AppConfig {
    @Bean
    @Lazy // 方法级别懒加载
    public ExpensiveBean expensiveBean() {
        return new ExpensiveBean();
    }
}
```
**XML配置**：
```xml
<bean id="lazyBean" class="com.example.LazyBean" lazy-init="true"/>
```
**应用场景**：
- 初始化耗时的Bean
- 不常用的功能模块
- 测试环境快速启动
**注意事项**：懒加载可能延迟发现配置错误，直到实际使用时才暴露问题。
###### 12. 如何在 Spring 中注入集合类型？
Spring支持注入多种集合类型，包括List、Set、Map和Properties。
**集合注入示例**：
```java
@Component
public class CollectionInjection {
    // 注入同类型Bean的List，按Order排序或自然顺序
    @Autowired
    private List<Validator> validators;
    
    // 注入Bean的Map，key为Bean名称
    @Autowired
    private Map<String, Processor> processors;
    
    // 注入具体值集合
    @Value("#{'${server.ports:8080,8081}'.split(',')}")
    private List<Integer> ports;
    
    // 注入Properties
    @Value("#{${app.config:{key1:'value1',key2:'value2'}}}")
    private Map<String, String> config;
}
```
**XML配置方式**：
```xml
<bean id="collectionBean" class="com.example.CollectionBean">
    <property name="list">
        <list>
            <value>first</value>
            <ref bean="someBean"/>
        </list>
    </property>
    <property name="map">
        <map>
            <entry key="key1" value="value1"/>
            <entry key="key2" value-ref="someBean"/>
        </map>
    </property>
</bean>
```
**类型匹配规则**：当注入Bean集合时，Spring会自动收集容器中所有匹配类型的Bean。使用`@Order`注解可以控制List中Bean的顺序。