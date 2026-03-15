# 04、Maven 命令与插件

## Q1. Maven 常用命令有哪些？

```bash
# ==================== 基础生命周期命令 ====================
mvn clean                      # 清理 target/ 目录
mvn compile                    # 编译主代码
mvn test                       # 运行测试
mvn package                    # 打包（jar/war）
mvn install                    # 打包并安装到本地仓库
mvn deploy                     # 打包并发布到远程仓库

# ==================== 常用组合 ====================
mvn clean package              # 清理后重新打包（最常用）
mvn clean install -DskipTests  # 跳过测试装本地仓库
mvn clean package -Pproduction # 激活 production profile

# ==================== 依赖分析 ====================
mvn dependency:tree            # 查看完整依赖树
mvn dependency:tree -Dincludes=org.slf4j:*  # 过滤特定依赖
mvn dependency:analyze         # 分析未使用和缺失的依赖
mvn dependency:list            # 所有依赖列表

# ==================== 其他有用命令 ====================
mvn help:effective-pom         # 查看最终生效的完整 POM（含继承和默认值）
mvn help:effective-settings    # 查看生效的 settings.xml
mvn versions:display-dependency-updates  # 显示可升级的依赖版本
mvn site                       # 生成项目文档站点
```

---

## Q2. `mvn package`、`mvn install`、`mvn deploy` 的区别？

```
mvn package  → 在 target/ 下生成 jar/war（只在本项目目录）
    ↓
mvn install  → package + 复制 jar 到 ~/.m2/repository/（本机可用）
    ↓
mvn deploy   → install + 上传到远程仓库（团队可用）
```

**使用场景**：
- `package`：只是想生成部署包，不需要本地仓库中
- `install`：多模块项目中，先 install 子模块，再构建依赖它的其他模块
- `deploy`：CI/CD 流程中，通过 Jenkins 将稳定版本发布给整个团队

---

## Q3. Maven 插件体系是什么？

Maven 的所有实际工作都由插件完成，生命周期只是一套调度框架：

```
阶段（Phase）→ 绑定 → 插件目标（Plugin Goal）
compile      →       maven-compiler-plugin:compile
test         →       maven-surefire-plugin:test
package(jar) →       maven-jar-plugin:jar
package(war) →       maven-war-plugin:war
install      →       maven-install-plugin:install
deploy       →       maven-deploy-plugin:deploy
```

**直接执行插件目标（不走生命周期）**：
```bash
mvn dependency:tree         # 直接执行 dependency 插件的 tree 目标
mvn exec:java -Dexec.mainClass=com.example.Main  # 运行 Java 程序
```

---

## Q4. Maven 常用插件有哪些？

```xml
<!-- maven-compiler-plugin — 编译，最重要 -->
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-compiler-plugin</artifactId>
    <configuration>
        <source>17</source>
        <target>17</target>
        <encoding>UTF-8</encoding>
    </configuration>
</plugin>

<!-- maven-surefire-plugin — 运行单元测试 -->
<plugin>
    <artifactId>maven-surefire-plugin</artifactId>
    <configuration>
        <skipTests>false</skipTests>
        <includes>
            <include>**/*Test.java</include>
        </includes>
    </configuration>
</plugin>

<!-- spring-boot-maven-plugin — 打 Spring Boot 可执行 jar -->
<plugin>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-maven-plugin</artifactId>
    <configuration>
        <excludes>
            <exclude>
                <groupId>org.projectlombok</groupId>
                <artifactId>lombok</artifactId>
            </exclude>
        </excludes>
    </configuration>
</plugin>

<!-- maven-shade-plugin — 打 fat jar（将所有依赖打进一个jar）-->
<plugin>
    <artifactId>maven-shade-plugin</artifactId>
</plugin>

<!-- maven-assembly-plugin — 灵活的打包工具 -->
<plugin>
    <artifactId>maven-assembly-plugin</artifactId>
</plugin>

<!-- maven-source-plugin — 生成源码包 -->
<plugin>
    <artifactId>maven-source-plugin</artifactId>
    <executions>
        <execution>
            <id>attach-sources</id>
            <goals><goal>jar</goal></goals>
        </execution>
    </executions>
</plugin>

<!-- versions-maven-plugin — 版本管理 -->
<plugin>
    <groupId>org.codehaus.mojo</groupId>
    <artifactId>versions-maven-plugin</artifactId>
</plugin>
```

---

## Q5. 如何跳过测试、指定测试？

```bash
# 跳过测试
mvn package -DskipTests                    # 跳过测试（不编译也不运行）
mvn package -Dmaven.test.skip=true         # 同上（完整参数）

# 运行指定测试类
mvn test -Dtest=UserServiceTest
mvn test -Dtest=UserServiceTest,OrderServiceTest  # 多个类
mvn test -Dtest=UserServiceTest#testCreate        # 指定方法

# 运行指定模式的测试
mvn test -Dtest="*ServiceTest"    # 通配符

# 只运行集成测试（需要配 failsafe 插件）
mvn verify
```

---

## Q6. maven-compiler-plugin 如何配置 Java 版本？

```xml
<!-- 方式1：在 properties 中配置（推荐，简洁）-->
<properties>
    <maven.compiler.source>17</maven.compiler.source>
    <maven.compiler.target>17</maven.compiler.target>
    <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
</properties>

<!-- 方式2：在 plugin 中配置（更灵活）-->
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-compiler-plugin</artifactId>
    <version>3.11.0</version>
    <configuration>
        <release>17</release>   <!-- JDK 9+ 推荐用 release 替代 source+target -->
    </configuration>
</plugin>
```

> **追问：`--release` 和 `source/target` 的区别？**
> `--release N` 是 JDK 9 引入的新选项，它不仅设置了 source 和 target，还确保使用当时 N 版本的 API（防止用了高版本 JDK 的 API 但 target 设了低版本）。推荐 JDK 9+ 项目用 `<release>17</release>` 替代 `<source>17</source><target>17</target>`。
