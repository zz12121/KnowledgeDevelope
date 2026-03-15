# SpringBoot 测试

## 一、测试体系概览

Spring Boot 测试基于 JUnit 5 和 Spring TestContext 框架构建，`spring-boot-starter-test` 传递引入完整工具链：JUnit 5、Mockito、AssertJ、JSONPath 等。

**测试注解分层（从重到轻）：**

| 注解 | 加载范围 | 适用场景 |
|------|----------|----------|
| `@SpringBootTest` | 完整应用上下文 | 集成测试、端到端测试 |
| `@WebMvcTest` | 仅 Web 相关组件 | Controller 层测试 |
| `@DataJpaTest` | 仅 JPA 相关组件 | Repository 层测试 |
| `@JsonTest` | 仅 JSON 序列化组件 | JSON 序列化测试 |
| `@MockitoExtension` | 无 Spring 上下文 | 纯单元测试 |

**核心原则：** 测试粒度越细越快，优先选择轻量级切片测试注解，避免每个测试都加载完整上下文。

---

## 二、@SpringBootTest 详解

`@SpringBootTest` 是组合注解，包含 `@BootstrapWith(SpringBootTestContextBootstrapper)` 和 `@ExtendWith(SpringExtension)`，从 `@SpringBootApplication` 主类开始执行完整的组件扫描和自动配置。

**四种 Web 环境模式：**

```java
// MOCK（默认）：模拟 Servlet 环境，不启动真实服务器
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.MOCK)

// RANDOM_PORT：启动嵌入式服务器，随机分配端口，真实 HTTP 测试
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
// 配合 @LocalServerPort 获取端口

// DEFINED_PORT：使用配置文件中定义的端口

// NONE：不提供 Web 环境，纯 Spring 上下文
```

**上下文缓存：** 相同配置的测试类共享同一个 Spring 上下文，避免重复初始化。`@DirtiesContext` 强制刷新缓存。

---

## 三、Mockito Mock 测试

**三种 Mock 使用方式：**

**方式一：纯 Mockito 单元测试（最轻量，推荐）：**

```java
@ExtendWith(MockitoExtension.class)
class UserServiceTest {

    @Mock
    private UserRepository userRepository;

    @InjectMocks
    private UserService userService;

    @Test
    void getUser_shouldReturnUser() {
        when(userRepository.findById(1L)).thenReturn(Optional.of(new User("John")));

        User result = userService.getUser(1L);

        assertThat(result.getName()).isEqualTo("John");
        verify(userRepository, times(1)).findById(1L);
    }
}
```

**方式二：Spring Boot 集成 Mock（@MockBean）：**

```java
@SpringBootTest
class ServiceIntegrationTest {

    @MockBean  // 注册到 Spring 上下文，替换同类型 Bean
    private ExternalService externalService;

    @Autowired
    private MainService mainService;

    @Test
    void testWithMockBean() {
        when(externalService.callApi()).thenReturn("mocked response");
        String result = mainService.processWithExternal();
        assertThat(result).contains("mocked");
    }
}
```

**@Mock vs @MockBean：**

- `@Mock`：Mockito 原生注解，不注入 Spring 上下文，速度快
- `@MockBean`：Spring Boot 注解，Mock 对象注册到 Spring 上下文，会重新加载上下文

**高级 Mockito 技巧：**

```java
// 异常模拟
when(userRepository.findById(999L)).thenThrow(new RuntimeException("not found"));

// 参数匹配器
when(userRepository.findById(anyLong())).thenReturn(defaultUser);
when(userRepository.findByEmail(eq("test@example.com"))).thenReturn(user);
when(userRepository.findByAge(argThat(age -> age > 18))).thenReturn(adults);

// 验证无多余交互
verifyNoMoreInteractions(userRepository);
```

---

## 四、Controller 层测试

基于 `@WebMvcTest` + `MockMvc`，模拟 Servlet 容器环境，不启动真实服务器。

```java
@WebMvcTest(UserController.class)  // 只加载 Web 相关组件，速度快
class UserControllerTest {

    @Autowired
    private MockMvc mockMvc;

    @MockBean
    private UserService userService;  // Mock 掉 Service 依赖

    @Test
    void getUser_shouldReturnUser() throws Exception {
        when(userService.getUser(1L)).thenReturn(new User(1L, "John"));

        mockMvc.perform(get("/users/1").accept(MediaType.APPLICATION_JSON))
            .andDo(print())
            .andExpect(status().isOk())
            .andExpect(jsonPath("$.id").value(1))
            .andExpect(jsonPath("$.name").value("John"));
    }

    @Test
    void createUser_shouldReturnCreated() throws Exception {
        mockMvc.perform(post("/users")
                .contentType(MediaType.APPLICATION_JSON)
                .content("""{"name":"John","email":"john@example.com"}"""))
            .andExpect(status().isCreated())
            .andExpect(header().string("Location", containsString("/users/")));
    }

    @Test
    void getUser_notFound_shouldReturn404() throws Exception {
        when(userService.getUser(999L)).thenThrow(new UserNotFoundException("not found"));

        mockMvc.perform(get("/users/999"))
            .andExpect(status().isNotFound())
            .andExpect(jsonPath("$.error").value("USER_NOT_FOUND"));
    }
}
```

**带安全上下文的测试：**

```java
mockMvc.perform(get("/profile")
        .with(SecurityMockMvcRequestPostProcessors.user("admin").roles("ADMIN")))
    .andExpect(status().isOk());
```

---

## 五、Service 层测试

**策略选择：**

- 业务逻辑复杂 → 纯 Mockito 单元测试，快速隔离
- 需要验证数据访问逻辑 → `@DataJpaTest` 集成测试

**事务性测试（自动回滚）：**

```java
@SpringBootTest
@Transactional  // 测试方法结束后自动回滚，不污染数据库
class UserServiceTxTest {

    @Autowired
    private UserService userService;

    @Test
    void testWithinTransaction() {
        User user = userService.createUser(new UserCreateRequest("Test", "test@example.com"));
        assertThat(userService.getUser(user.getId())).isPresent();
    }
    // 方法结束，事务自动回滚
}
```

**异步方法测试：**

```java
@Test
void processAsync_shouldComplete() {
    CompletableFuture<String> future = asyncService.processAsync("data");

    assertThat(future)
        .succeedsWithin(5, TimeUnit.SECONDS)
        .isEqualTo("processed_data");
}
```

---

## 六、Repository 层测试

`@DataJpaTest` 自动配置内存数据库（H2）和 JPA，不加载 Web 层，速度快。

```java
@DataJpaTest
class UserRepositoryTest {

    @Autowired
    private TestEntityManager entityManager;  // JPA 测试工具

    @Autowired
    private UserRepository userRepository;

    @BeforeEach
    void setUp() {
        entityManager.persistFlushFind(new User("John", "john@example.com"));
    }

    @Test
    void findByEmail_shouldReturnUser() {
        Optional<User> found = userRepository.findByEmail("john@example.com");
        assertThat(found).isPresent();
    }

    @Test
    void existsByEmail_shouldReturnTrue() {
        assertThat(userRepository.existsByEmail("john@example.com")).isTrue();
    }
}
```

**使用 SQL 脚本初始化测试数据：**

```java
@Sql("/test-data/users.sql")  // 方法执行前运行 SQL 脚本
@Test
void testWithSqlData() {
    List<User> users = userRepository.findAll();
    assertThat(users).hasSize(5);
}
```

**使用真实数据库（Testcontainers）：**

```java
@SpringBootTest
@Testcontainers
class RealDatabaseTest {

    @Container
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:13");

    @DynamicPropertySource
    static void configureProperties(DynamicPropertyRegistry registry) {
        registry.add("spring.datasource.url", postgres::getJdbcUrl);
        registry.add("spring.datasource.username", postgres::getUsername);
        registry.add("spring.datasource.password", postgres::getPassword);
    }
}
```

---

## 七、集成测试

集成测试验证多个组件协同工作的正确性，使用真实或接近真实的环境。

```java
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
class ApplicationIntegrationTest {

    @Autowired
    private TestRestTemplate restTemplate;  // 支持真实 HTTP 调用

    @LocalServerPort
    private int serverPort;

    @Test
    void fullWorkflow_shouldWork() {
        ResponseEntity<User> response = restTemplate.getForEntity(
            "http://localhost:" + serverPort + "/users/1", User.class);

        assertThat(response.getStatusCode()).isEqualTo(HttpStatus.OK);
    }
}
```

---

## 八、响应式编程测试

Reactor 测试使用 `StepVerifier`：

```java
@Test
void fluxOperation_shouldEmitItems() {
    Flux<String> result = reactiveUserService.getUsers();

    StepVerifier.create(result)
        .expectNext("user1", "user2", "user3")
        .verifyComplete();
}
```

---

## 关联面试题

- 📝 [[10_Developlanguage/005_Spring/03_SpringBootSubject/09、测试]]
