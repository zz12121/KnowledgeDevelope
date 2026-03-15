# 07、Maven 高级特性

## Q1. SNAPSHOT 和 RELEASE 版本的区别？

```
SNAPSHOT（快照版本）：版本号以 -SNAPSHOT 结尾，如 1.0.0-SNAPSHOT
RELEASE（发布版本）：版本号不带后缀，如 1.0.0 或 1.0.0.RELEASE
```

**核心区别**：

| 特性 | SNAPSHOT | RELEASE |
|------|---------|---------|
| 更新策略 | 每次构建检查远程是否有新版本 | 一旦下载到本地，永不重新下载 |
| 适用场景 | 开发中的库，频繁变更 | 稳定的发布版本 |
| 生产环境 | ❌ 不允许（不可重现）| ✅ 必须用 RELEASE |
| 私服仓库 | snapshots 仓库 | releases 仓库 |
| 时间戳 | 实际存储带时间戳（如 1.0.0-20260315.143000-1）| 固定版本 |

```bash
# 强制更新 SNAPSHOT 版本
mvn clean install -U      # -U 强制从远程仓库更新
```

**实际工作流**：开发阶段用 `1.0.0-SNAPSHOT`，发布到测试/生产时用 `mvn versions:set -DnewVersion=1.0.0` 去掉 SNAPSHOT，然后 `mvn deploy` 发布到 releases 仓库。

---

## Q2. 如何实现多环境配置（Profile + 资源过滤）？

```xml
<!-- pom.xml -->
<profiles>
    <profile>
        <id>dev</id>
        <activation><activeByDefault>true</activeByDefault></activation>
        <properties>
            <env>dev</env>
        </properties>
    </profile>
    <profile>
        <id>prod</id>
        <properties>
            <env>prod</env>
        </properties>
    </profile>
</profiles>

<build>
    <resources>
        <resource>
            <directory>src/main/resources</directory>
            <filtering>true</filtering>   <!-- 开启资源文件变量替换 -->
        </resource>
        <!-- 根据 profile 加载不同目录的配置 -->
        <resource>
            <directory>src/main/resources-${env}</directory>
            <filtering>false</filtering>
        </resource>
    </resources>
</build>
```

目录结构：
```
src/main/
├── resources/              ← 公共配置
│   └── application.properties
├── resources-dev/          ← 开发环境配置
│   └── application-db.properties
└── resources-prod/         ← 生产环境配置
    └── application-db.properties
```

```bash
mvn clean package -Pprod    # 打包生产环境版本
```

**Spring Boot 项目更简单的方式**：直接用 `application-dev.yml`、`application-prod.yml`，通过 `spring.profiles.active` 切换，不需要 Maven Profile（除非要控制依赖差异）。

---

## Q3. Maven 如何处理资源文件？

资源文件（`src/main/resources/`）默认会被复制到 `target/classes/`。

**资源过滤（变量替换）**：
```xml
<build>
    <resources>
        <resource>
            <directory>src/main/resources</directory>
            <filtering>true</filtering>
        </resource>
    </resources>
</build>
```

```properties
# application.properties（注意 @...@ 是 Spring Boot starter parent 的默认分隔符）
spring.profiles.active=@env@
app.version=@project.version@
```

**注意**：`filtering=true` 会处理 `.properties` 文件中的 `${variable}` 或 `@variable@`，但如果资源文件是二进制格式（图片、字体、Word等）需要排除，否则可能损坏文件：
```xml
<resources>
    <resource>
        <directory>src/main/resources</directory>
        <filtering>true</filtering>
        <excludes>
            <exclude>**/*.png</exclude>
            <exclude>**/*.docx</exclude>
        </excludes>
    </resource>
    <resource>
        <directory>src/main/resources</directory>
        <filtering>false</filtering>
        <includes>
            <include>**/*.png</include>
        </includes>
    </resource>
</resources>
```

---

## Q4. Maven Wrapper 是什么？

Maven Wrapper（`mvnw`）类似于 Gradle Wrapper，可以让项目携带指定版本的 Maven，无需在本机安装：

```bash
# 生成 wrapper（需要项目已有 Maven）
mvn wrapper:wrapper                           # 使用当前 Maven 版本
mvn wrapper:wrapper -Dmaven=3.9.6             # 指定版本

# 生成的文件
.mvn/wrapper/maven-wrapper.properties         # 指定 Maven 下载地址
.mvn/wrapper/maven-wrapper.jar                # wrapper 引导程序
mvnw                                           # Linux/Mac 脚本
mvnw.cmd                                       # Windows 脚本

# 使用（会自动下载指定版本的 Maven）
./mvnw clean package
```

**好处**：
- CI/CD 环境不需要预装 Maven，`./mvnw` 自动下载正确版本
- 保证所有人用同一版本 Maven，避免版本差异导致的构建问题
- Spring Initializr 生成的项目默认包含 Maven Wrapper

---

## Q5. Maven archetype 是什么？

Archetype 是项目模板系统，用于快速生成符合规范的项目骨架：

```bash
# 交互式创建（会提示选择 archetype）
mvn archetype:generate

# 非交互式创建 Spring Boot 项目
mvn archetype:generate \
  -DgroupId=com.example \
  -DartifactId=my-project \
  -DarchetypeGroupId=org.springframework.boot \
  -DarchetypeArtifactId=spring-boot-sample-web-archetype \
  -DinteractiveMode=false

# 常用 archetype
# maven-archetype-quickstart  最简单的 Java 项目骨架
# maven-archetype-webapp       Java Web 项目（无 Spring）
```

**实际工作中**：大公司通常会创建自己的 archetype（内部脚手架），包含公司规范的目录结构、通用依赖、日志配置、代码规范检查插件等，新建项目时用内部 archetype 一键生成，确保所有项目符合规范。

---

## Q6. 如何使用 Maven 进行测试？

```xml
<!-- maven-surefire-plugin — 单元测试 -->
<plugin>
    <artifactId>maven-surefire-plugin</artifactId>
    <configuration>
        <!-- 测试报告格式 -->
        <reportFormat>xml</reportFormat>
        <!-- JVM 参数（如内存配置）-->
        <argLine>-Xmx512m -Dfile.encoding=UTF-8</argLine>
        <!-- 并行测试 -->
        <parallel>methods</parallel>
        <threadCount>4</threadCount>
        <!-- 失败时继续（不中断构建）-->
        <failIfNoTests>false</failIfNoTests>
    </configuration>
</plugin>

<!-- maven-failsafe-plugin — 集成测试（verify 阶段执行）-->
<plugin>
    <artifactId>maven-failsafe-plugin</artifactId>
    <executions>
        <execution>
            <goals>
                <goal>integration-test</goal>
                <goal>verify</goal>
            </goals>
        </execution>
    </executions>
</plugin>
```

**测试类命名约定**（surefire 默认识别）：
- `**/*Test.java`、`**/*Tests.java`
- `**/Test*.java`
- `**/*TestCase.java`

```bash
mvn test                           # 运行单元测试
mvn verify                         # 运行单元测试 + 集成测试
mvn test -Dtest=UserServiceTest    # 运行指定测试
mvn test -DfailIfNoTests=false     # 没有测试时不报错
```

> **追问：JaCoCo 代码覆盖率如何集成？**
> ```xml
> <plugin>
>     <groupId>org.jacoco</groupId>
>     <artifactId>jacoco-maven-plugin</artifactId>
>     <version>0.8.11</version>
>     <executions>
>         <execution>
>             <goals><goal>prepare-agent</goal></goals>
>         </execution>
>         <execution>
>             <id>report</id>
>             <phase>test</phase>
>             <goals><goal>report</goal></goals>
>         </execution>
>         <!-- 可选：覆盖率低于阈值时构建失败 -->
>         <execution>
>             <id>check</id>
>             <goals><goal>check</goal></goals>
>             <configuration>
>                 <rules>
>                     <rule>
>                         <limits>
>                             <limit>
>                                 <counter>INSTRUCTION</counter>
>                                 <value>COVEREDRATIO</value>
>                                 <minimum>0.70</minimum>  <!-- 70% 最低覆盖率 -->
>                             </limit>
>                         </limits>
>                     </rule>
>                 </rules>
>             </configuration>
>         </execution>
>     </executions>
> </plugin>
> ```
> 执行 `mvn verify` 后，报告生成在 `target/site/jacoco/index.html`。
