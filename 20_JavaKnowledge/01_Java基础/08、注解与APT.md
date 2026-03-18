---
tags:
  - Java/基础
  - Java/注解
aliases:
  - Annotation
  - APT注解处理器
  - Lombok原理
date: 2026-03-18
---

# 注解与 APT

> **核心关键词**：@interface、元注解、RetentionPolicy、ElementType、注解处理器、APT、编译期代码生成

---

## 一、注解概述

**注解（Annotation）**：附加在代码元素（类、方法、字段等）上的元数据，本身不影响代码逻辑，但可被编译器、JVM 或框架在编译期 / 运行时读取并处理。

```
注解的三种使用场景：
  1. 编译期校验     → @Override、@SuppressWarnings（编译器直接处理）
  2. 编译期代码生成  → Lombok、MapStruct、Dagger（APT 处理）
  3. 运行时处理     → Spring、JUnit、Jackson（反射读取）
```

---

## 二、自定义注解

```java
// 注解定义语法（本质是特殊接口）
public @interface MyAnnotation {
    // 注解元素（无参方法），可设 default 值
    String value() default "";
    int order() default 0;
    String[] tags() default {};
    Class<?> handler() default Void.class;
}

// 使用
@MyAnnotation(value = "test", order = 1, tags = {"a", "b"})
public class MyClass { }

// 只有一个 value 元素时可省略 key
@MyAnnotation("test")
public class MyClass { }
```

---

## 三、元注解（Meta-Annotation）

元注解是用来修饰注解本身的注解，共 5 个：

### 3.1 @Retention — 保留策略（最重要）

```java
@Retention(RetentionPolicy.SOURCE)   // 仅源码，编译后丢弃（@Override、@SuppressWarnings）
@Retention(RetentionPolicy.CLASS)    // 保留到 .class 文件，JVM 加载时丢弃（默认）
@Retention(RetentionPolicy.RUNTIME)  // 运行时保留，可通过反射读取（Spring、JUnit 等框架注解）
```

> ⚠️ **关键**：只有 `RUNTIME` 保留策略的注解才能在运行时通过反射读取到！

### 3.2 @Target — 作用目标

```java
@Target({
    ElementType.TYPE,             // 类、接口、枚举
    ElementType.FIELD,            // 字段（含枚举常量）
    ElementType.METHOD,           // 方法
    ElementType.PARAMETER,        // 方法参数
    ElementType.CONSTRUCTOR,      // 构造器
    ElementType.LOCAL_VARIABLE,   // 局部变量
    ElementType.ANNOTATION_TYPE,  // 注解类型（元注解）
    ElementType.PACKAGE,          // 包
    ElementType.TYPE_PARAMETER,   // 泛型参数（JDK 8）
    ElementType.TYPE_USE,         // 任何类型使用处（JDK 8，如 List<@NotNull String>）
    ElementType.MODULE,           // 模块（JDK 9）
    ElementType.RECORD_COMPONENT  // Record 组件（JDK 16）
})
```

### 3.3 @Documented、@Inherited、@Repeatable

```java
@Documented    // 该注解出现在 JavaDoc 中
@Inherited     // 子类自动继承父类的该注解（只作用于类上的注解）
@Repeatable(Schedules.class)  // 允许同一位置重复使用，需配套容器注解
public @interface Schedule {
    String cron();
}

@interface Schedules {  // 容器注解
    Schedule[] value();
}

// 使用 Repeatable
@Schedule(cron = "0 0 8 * * ?")
@Schedule(cron = "0 0 20 * * ?")
public void task() { }
```

---

## 四、运行时注解处理（反射）

```java
// 完整示例：简单的 ORM 字段映射
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.FIELD)
public @interface Column {
    String name();
    boolean nullable() default true;
    int length() default 255;
}

public class User {
    @Column(name = "user_name", nullable = false, length = 50)
    private String name;

    @Column(name = "age")
    private int age;
}

// 解析注解，生成 SQL
public static String buildCreateSQL(Class<?> clazz) {
    StringBuilder sb = new StringBuilder("CREATE TABLE (");
    for (Field field : clazz.getDeclaredFields()) {
        Column col = field.getAnnotation(Column.class);
        if (col != null) {
            sb.append(col.name())
              .append(" VARCHAR(").append(col.length()).append(")")
              .append(col.nullable() ? "" : " NOT NULL")
              .append(", ");
        }
    }
    sb.append(")");
    return sb.toString();
}
```

---

## 五、APT（Annotation Processing Tool）

### 5.1 APT 是什么

APT 是 Java 编译器内置的注解处理框架，允许在**编译阶段**扫描源码中的注解，并生成新的 Java 源文件（不修改原文件）。

```
编译流程：
  Java 源码 → javac 解析 → APT 处理器运行 → 生成新源码 → 再次编译 → .class 文件
```

**典型应用**：
- **Lombok**：`@Data`、`@Builder` → 生成 getter/setter/builder
- **MapStruct**：`@Mapper` → 生成类型映射代码
- **Dagger2**：`@Component`、`@Module` → 生成依赖注入代码
- **AutoService**：`@AutoService` → 自动生成 SPI 配置文件
- **Room（Android）**：`@Entity`、`@Dao` → 生成数据库访问代码

### 5.2 自定义 APT 处理器

```java
// 1. 引入依赖（auto-service 简化处理器注册）
// implementation "com.google.auto.service:auto-service:1.1.1"

// 2. 实现处理器
@AutoService(Processor.class)          // 自动注册到 SPI
@SupportedAnnotationTypes("com.example.Builder")  // 处理哪些注解
@SupportedSourceVersion(SourceVersion.RELEASE_17)
public class BuilderProcessor extends AbstractProcessor {

    @Override
    public boolean process(Set<? extends TypeElement> annotations,
                           RoundEnvironment roundEnv) {
        for (TypeElement annotation : annotations) {
            for (Element element : roundEnv.getElementsAnnotatedWith(annotation)) {
                if (element.getKind() == ElementKind.CLASS) {
                    TypeElement classElement = (TypeElement) element;
                    generateBuilder(classElement);  // 生成 Builder 类
                }
            }
        }
        return true;  // true = 注解被消费，不传递给下一个处理器
    }

    private void generateBuilder(TypeElement classElement) {
        // 使用 JavaPoet 库优雅地生成 Java 代码
        String className = classElement.getSimpleName() + "Builder";
        // ... 构建 TypeSpec、MethodSpec，写入文件
        try {
            JavaFileObject file = processingEnv.getFiler()
                .createSourceFile(className);
            // 写入生成的源码
        } catch (IOException e) {
            processingEnv.getMessager()
                .printMessage(Diagnostic.Kind.ERROR, e.getMessage());
        }
    }
}

// 3. 手动注册（不用 AutoService 时）
// 在 resources/META-INF/services/javax.annotation.processing.Processor 文件中
// 写入处理器全限定类名：com.example.BuilderProcessor
```

### 5.3 APT 核心 API

```java
// ProcessingEnvironment 提供的工具
processingEnv.getFiler()       // Filer：创建新源文件
processingEnv.getMessager()    // Messager：输出编译时警告/错误
processingEnv.getElementUtils() // Elements：元素操作工具
processingEnv.getTypeUtils()   // Types：类型操作工具

// 常用 Element 类型
TypeElement     // 类或接口
VariableElement // 字段或方法参数
ExecutableElement // 方法或构造器
PackageElement  // 包
```

---

## 六、注解 vs 配置文件 vs XML

| 维度 | 注解 | properties/yaml | XML |
|------|------|-----------------|-----|
| 类型安全 | ✅ 编译期校验 | ❌ 字符串 | ❌ 字符串 |
| 与代码耦合 | 高（侵入性） | 低 | 低 |
| 适合场景 | 固定配置、元数据描述 | 环境相关配置 | 复杂结构配置 |
| 运行时修改 | ❌ 不可更改 | ✅ 可外部化 | ✅ 可外部化 |

---

## 七、Lombok 底层原理深度解析

Lombok 并不是标准 APT，它直接操作 javac 的 **AST（抽象语法树）**，在编译期 **修改** 现有类的 AST，而非生成新文件。

```
标准 APT（JSR-269）：只能生成新文件，不能修改已有源码
Lombok：调用 com.sun.tools.javac 内部 API 修改 AST → 注入字段/方法

Lombok 编译流程：
  1. javac 解析源码生成 AST
  2. Lombok 的 AnnotationProcessor 运行
  3. 扫描 @Data / @Builder / @Slf4j 等注解
  4. 使用 javac Trees API 操作对应的 JCTree（javac 内部 AST 节点）
  5. 向类的 AST 注入 getter/setter/constructor/logger 等节点
  6. javac 继续编译（已注入节点的 AST）→ .class 文件

风险点：
  - 依赖 javac 内部 API（非公开 API），JDK 版本升级可能失效
  - 与某些 IDE / 编译工具可能不兼容（需要 IDE 安装 Lombok 插件）
  - @EqualsAndHashCode 在继承场景下可能产生预期外行为
  - 反射查找字段时可能找不到（生成的方法在字节码中，不在源码中）
```

```java
// Lombok 常用注解一览
@Data                // = @Getter + @Setter + @ToString + @EqualsAndHashCode + @RequiredArgsConstructor
@Builder             // 生成 Builder 模式（内部类 Builder + builder() 静态方法）
@Builder.Default     // Builder 字段的默认值
@SuperBuilder        // 支持继承的 Builder
@Value               // 不可变类（final 字段 + @Getter + @ToString + @EqualsAndHashCode）
@Slf4j               // 注入 Logger log = LoggerFactory.getLogger(Xxx.class);
@RequiredArgsConstructor  // 为 final 字段或 @NonNull 字段生成构造器（Spring 推荐注入方式）
@SneakyThrows        // 悄悄把 checked 异常包成 RuntimeException（慎用）
@Cleanup             // 自动在 finally 调用 close()（类似 try-with-resources）

// Spring + Lombok 推荐组合
@Service
@RequiredArgsConstructor  // 自动生成 final 字段的构造器注入
public class OrderService {
    private final OrderRepository orderRepo;  // final 触发构造器注入
    private final PaymentService paymentService;
    // 无需写 @Autowired 和构造函数
}
```

---

## 八、面试要点速查

| 问题 | 要点 |
|------|------|
| 注解的三种保留策略 | SOURCE（编译丢弃）/ CLASS（class 文件保留，JVM 丢弃）/ RUNTIME（运行时可反射读取） |
| 运行时注解如何读取 | `element.getAnnotation(Xxx.class)` 或 `element.getAnnotations()` |
| APT 与反射的区别 | APT 在**编译期**处理，无运行时开销；反射在**运行时**处理，有性能损耗 |
| Lombok 原理 | 并非标准 APT，而是用 javac 内部 API 直接**修改 AST**，注入代码到现有类 |
| @Repeatable 如何实现 | 需配套容器注解，通过 `getAnnotationsByType()` 获取重复注解数组 |
| 元注解有哪些 | @Retention、@Target、@Documented、@Inherited、@Repeatable |
| 标准 APT 的限制 | 只能生成新文件，不能修改已有源码（Lombok 为此绕过了标准API）|
| Spring @Required vs @Autowired | Spring 注解是 RUNTIME 保留，启动时通过反射扫描，不需要 APT |


---

**相关面试题** → [[../../10_Developlanguage/001_Java/01_JavaBaseSubject/13、注解|13、注解]]
