# 05、POM 文件配置

## Q1. POM 文件的完整结构是什么？

POM（Project Object Model）是 Maven 项目的核心配置文件：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 
                             http://maven.apache.org/xsd/maven-4.0.0.xsd">

    <modelVersion>4.0.0</modelVersion>

    <!-- 继承（可选）-->
    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>3.2.0</version>
    </parent>

    <!-- 项目坐标 -->
    <groupId>com.example</groupId>
    <artifactId>my-project</artifactId>
    <version>1.0.0-SNAPSHOT</version>
    <packaging>jar</packaging>    <!-- jar/war/pom -->

    <!-- 属性（变量）-->
    <properties>
        <java.version>17</java.version>
        <mysql.version>8.3.0</mysql.version>
    </properties>

    <!-- 依赖版本管理（不引入，只锁定版本）-->
    <dependencyManagement>...</dependencyManagement>

    <!-- 实际引入的依赖 -->
    <dependencies>...</dependencies>

    <!-- 构建配置 -->
    <build>
        <!-- 插件版本管理 -->
        <pluginManagement>...</pluginManagement>
        <!-- 实际使用的插件 -->
        <plugins>...</plugins>
    </build>

    <!-- 多环境配置 -->
    <profiles>...</profiles>

    <!-- 发布目标仓库 -->
    <distributionManagement>...</distributionManagement>

    <!-- 多模块聚合（父 POM 中用）-->
    <modules>...</modules>
</project>
```

---

## Q2. POM 中的继承（Inheritance）是什么？

Maven 中 POM 可以继承父 POM，子 POM 获得父 POM 的所有配置（可覆盖）：

```xml
<!-- 子模块 pom.xml -->
<parent>
    <groupId>com.example</groupId>
    <artifactId>parent-project</artifactId>
    <version>1.0.0</version>
    <relativePath>../pom.xml</relativePath>  <!-- 父POM路径，默认../pom.xml -->
</parent>
```

**可继承的内容**：
- `properties`、`dependencyManagement`、`dependencies`
- `pluginManagement`、`plugins`
- `repositories`

**Spring Boot 项目为什么要继承 `spring-boot-starter-parent`？**
因为它提供了：所有 Spring 生态依赖的版本管理、compiler 配置（UTF-8 + Java版本）、resource filtering 配置、可执行 jar 打包配置等，让你直接 `<dependency>` 不用写 version。

---

## Q3. POM 中的聚合（Aggregation）是什么？

聚合让你用一条命令构建多个模块：

```xml
<!-- 父 POM（packaging 必须是 pom）-->
<groupId>com.example</groupId>
<artifactId>parent</artifactId>
<version>1.0.0</version>
<packaging>pom</packaging>

<modules>
    <module>common</module>      <!-- 子模块目录名 -->
    <module>service</module>
    <module>web</module>
</modules>
```

```bash
# 在父 POM 目录执行，自动按依赖顺序构建所有子模块
mvn clean install
```

**继承 vs 聚合**：
- 聚合：父 POM 知道有哪些子模块（modules 配置），用于批量构建
- 继承：子 POM 知道有哪个父 POM（parent 配置），用于配置继承
- 两者可以同时使用，实际项目中通常在同一个父 POM 上都有

---

## Q4. pluginManagement 和 plugins 的区别？

与 `dependencyManagement` / `dependencies` 类比：

```xml
<!-- pluginManagement：只声明版本和默认配置，不实际绑定 -->
<build>
    <pluginManagement>
        <plugins>
            <plugin>
                <artifactId>maven-surefire-plugin</artifactId>
                <version>3.2.5</version>
                <configuration>
                    <skipTests>false</skipTests>
                </configuration>
            </plugin>
        </plugins>
    </pluginManagement>
</build>
```

子模块想用时，只需声明 artifactId，不用重复写版本和配置：
```xml
<build>
    <plugins>
        <plugin>
            <artifactId>maven-surefire-plugin</artifactId>
            <!-- version 和 configuration 继承自 pluginManagement -->
        </plugin>
    </plugins>
</build>
```

---

## Q5. 如何在 POM 中定义和使用属性（properties）？

```xml
<properties>
    <!-- 内置属性 -->
    <maven.compiler.source>17</maven.compiler.source>
    <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>

    <!-- 自定义属性（统一管理版本号）-->
    <spring-cloud.version>2023.0.0</spring-cloud.version>
    <mybatis-plus.version>3.5.5</mybatis-plus.version>
    <swagger.version>2.2.19</swagger.version>
</properties>

<!-- 使用属性 -->
<dependency>
    <groupId>com.baomidou</groupId>
    <artifactId>mybatis-plus-boot-starter</artifactId>
    <version>${mybatis-plus.version}</version>   <!-- ${属性名} 引用 -->
</dependency>
```

**内置可用属性**：
- `${project.version}`：当前项目版本
- `${project.basedir}`：项目根目录（pom.xml 所在目录）
- `${project.build.directory}`：target 目录
- `${maven.home}`：Maven 安装目录
- `${env.PATH}`：环境变量

---

## Q6. BOM（Bill of Materials）是什么？如何使用？

BOM 是一种特殊的 pom，只有 `dependencyManagement`，专门用来统一管理一套相关依赖的版本：

```xml
<!-- 导入 BOM（在 dependencyManagement 中，scope=import type=pom）-->
<dependencyManagement>
    <dependencies>
        <!-- 导入 Spring Cloud BOM，自动管理所有 Spring Cloud 组件版本 -->
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-dependencies</artifactId>
            <version>2023.0.0</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
    </dependencies>
</dependencyManagement>

<!-- 之后使用 Spring Cloud 组件就不用写版本了 -->
<dependencies>
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-openfeign</artifactId>
        <!-- 无需版本，由 BOM 管理 -->
    </dependency>
</dependencies>
```

**好处**：引入一整套框架时，所有组件版本由框架官方保证兼容性，不用自己逐个指定和测试。

---

## Q7. Maven Profiles 是什么？如何使用？

Profile 允许根据不同条件激活不同的配置（构建环境、JDK 版本、操作系统等）：

```xml
<profiles>
    <!-- 开发环境 -->
    <profile>
        <id>dev</id>
        <activation>
            <activeByDefault>true</activeByDefault>  <!-- 默认激活 -->
        </activation>
        <properties>
            <env>dev</env>
            <log.level>DEBUG</log.level>
        </properties>
        <dependencies>
            <dependency>
                <groupId>com.h2database</groupId>
                <artifactId>h2</artifactId>
            </dependency>
        </dependencies>
    </profile>

    <!-- 生产环境 -->
    <profile>
        <id>production</id>
        <properties>
            <env>production</env>
            <log.level>WARN</log.level>
        </properties>
        <build>
            <plugins>
                <plugin>
                    <artifactId>maven-compiler-plugin</artifactId>
                    <configuration>
                        <optimize>true</optimize>  <!-- 开启编译优化 -->
                    </configuration>
                </plugin>
            </plugins>
        </build>
    </profile>
</profiles>
```

```bash
# 激活指定 profile
mvn package -Pproduction        # 激活 production profile
mvn package -Pdev,monitoring    # 激活多个

# 查看哪些 profile 被激活
mvn help:active-profiles
```

> **追问：Profile 和 Spring Boot 的多环境配置有什么关系？**
> Maven Profile 控制**构建时**的差异（哪些依赖、哪些编译选项）；Spring Boot 的 `application-{env}.yml` 控制**运行时**的差异（数据库地址、日志级别等）。通常两者配合使用：Maven Profile 设置 `@spring.profiles.active@=prod`，资源过滤后注入到 `application.properties` 中，Spring Boot 启动时读取对应环境的配置文件。
