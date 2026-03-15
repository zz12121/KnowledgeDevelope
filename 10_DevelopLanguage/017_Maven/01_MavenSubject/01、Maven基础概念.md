# 01、Maven 基础概念

## Q1. Maven 是什么？

Maven 是 Apache 基金会开源的**项目管理和构建工具**，专为 Java 项目设计。它的核心价值是用一个 `pom.xml` 配置文件统一管理项目的依赖、构建、测试和发布流程。

**核心能力三句话**：
- **依赖管理**：声明依赖，Maven 自动从仓库下载 jar 包和传递性依赖
- **标准化构建**：compile → test → package → install → deploy 生命周期统一
- **项目信息管理**：版本号、开发者、SCM 地址等元数据集中管理

实际开发中用 Maven 最主要的原因就一个：**再也不用手动管理一堆 jar 包**，依赖冲突也能有规则可循。

> **追问：Maven 和 Ant 的区别？**
> Ant 是纯构建工具，类似于 Makefile，只负责执行你写的构建步骤，没有约定目录结构，没有依赖管理；Maven 是"约定优于配置"，定义了标准目录结构和生命周期，内置依赖管理，你只需要声明"要什么"，Maven 负责"怎么做"。Ant 灵活但繁琐，Maven 标准化但有约束。

---

## Q2. 为什么项目选用 Maven 构建？

几个实际原因：

1. **生态成熟**：Spring、MyBatis 等主流框架都发布在 Maven Central，开箱即用
2. **团队协作**：所有人用同一套 pom.xml，依赖版本统一，告别"在我机器上能跑"
3. **CI/CD 集成**：`mvn package` 一条命令完成编译测试打包，天然适合 Jenkins/GitLab CI
4. **IDE 支持完美**：IntelliJ IDEA、Eclipse 等都原生支持 Maven 项目结构
5. **Spring Boot 推荐**：Spring Initializr 默认生成 Maven 项目

---

## Q3. Maven 的规约（约定优于配置）是什么？

Maven 定义了一套默认的项目结构，如果你遵循这套结构，就不需要写大量配置：

```
project/
├── pom.xml                    # 项目描述和配置
└── src/
    ├── main/
    │   ├── java/              # 生产代码
    │   └── resources/         # 生产资源文件（配置文件等）
    ├── test/
    │   ├── java/              # 测试代码
    │   └── resources/         # 测试资源文件
    └── filters/               # 资源过滤文件（可选）
target/                        # 构建输出目录（自动生成）
```

这套约定的好处：任何人拿到一个 Maven 项目，不看文档就知道代码在哪里、配置文件在哪里、测试在哪里。

---

## Q4. Maven 的生命周期有哪些？

Maven 有三套相互独立的生命周期：

**default（默认生命周期，最重要）**：
```
validate → compile → test → package → verify → install → deploy
   ↑校验      ↑编译     ↑测试   ↑打包      ↑验证      ↑装到       ↑发布到
   pom正确                           (集成测试) 本地仓库   远程仓库
```

**clean（清理生命周期）**：
```
pre-clean → clean → post-clean
              ↑删除 target/ 目录
```

**site（站点生命周期）**：
```
pre-site → site → post-site → site-deploy
            ↑生成项目文档站点
```

**关键点**：执行某个阶段会自动执行它之前的所有阶段。比如 `mvn package` 会先执行 validate → compile → test → package 所有步骤。

```bash
mvn clean package      # 先clean，再package（最常用）
mvn clean install -DskipTests  # 跳过测试装本地仓库
mvn deploy             # 发布到远程仓库
```

---

## Q5. Maven 的核心概念有哪些？

**POM（Project Object Model）**：`pom.xml` 文件，描述项目的一切——坐标、依赖、插件、构建配置。

**坐标（Coordinates）**：唯一标识一个构件的三元组：
```xml
<groupId>com.example</groupId>        <!-- 组织/公司 -->
<artifactId>my-project</artifactId>   <!-- 项目名 -->
<version>1.0.0</version>              <!-- 版本 -->
<packaging>jar</packaging>            <!-- 打包类型，默认jar -->
```

**仓库（Repository）**：存放构件的地方：
- 本地仓库：`~/.m2/repository/`（本机缓存）
- 远程仓库：Maven Central、私服（Nexus/Artifactory）

**依赖（Dependency）**：项目需要的外部 jar 包，Maven 自动下载并管理。

**插件（Plugin）**：Maven 的功能都靠插件实现，生命周期中的每个阶段都绑定了具体的插件目标（goal）来执行。

**Archetype**：项目模板，用于快速生成标准 Maven 项目骨架。

---

## Q6. Maven 的优缺点是什么？

**优点**：
- 依赖自动管理，传递性依赖自动处理
- 标准化项目结构，团队一致性强
- 丰富的中央仓库，生态完整
- 与主流 IDE 和 CI 工具无缝集成
- 多模块项目管理能力强

**缺点**：
- **XML 配置冗长**：pom.xml 动辄几百行，可读性差
- **构建速度**：全量构建大型项目慢（Gradle 的增量构建比 Maven 快）
- **灵活性受限**：约定过于死板，非标准场景配置复杂
- **版本冲突排查**：依赖冲突有时难以定位
- **学习曲线**：插件体系和生命周期概念需要时间消化

> **追问：Maven vs Gradle 如何选择？**
> 见第八章对比分析。简单说：**新项目**用 Gradle（更简洁、构建更快，Android 官方支持）；**存量 Java 项目**继续用 Maven（生态成熟、团队熟悉）；Spring Boot 两者都支持，Spring 官方文档两种都给示例。
