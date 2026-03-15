# 06、Maven 多模块项目

## Q1. 什么是 Maven 多模块项目？

多模块项目（Multi-Module Project）是将一个大型项目拆分成若干子模块，每个子模块是独立的 Maven 项目，有自己的 `pom.xml`，但由父 POM 统一管理和构建。

**典型的微服务/中台项目结构**：
```
my-platform/               ← 根模块（父 POM）
├── pom.xml                ← packaging=pom，管理所有子模块
├── common/                ← 公共工具模块
│   └── pom.xml
├── api/                   ← 对外接口定义（DTO/接口）
│   └── pom.xml
├── service-user/          ← 用户服务
│   └── pom.xml
├── service-order/         ← 订单服务
│   └── pom.xml
└── gateway/               ← 网关
    └── pom.xml
```

**好处**：
- 模块职责清晰，降低耦合
- 公共代码（common）只打包一次，各模块引用
- 可以独立编译某个子模块，提高构建速度
- 代码所有权清晰，便于团队分工

---

## Q2. 父模块 POM 应该包含什么？

```xml
<!-- parent/pom.xml -->
<groupId>com.example</groupId>
<artifactId>my-platform</artifactId>
<version>1.0.0-SNAPSHOT</version>
<packaging>pom</packaging>        <!-- 必须是 pom -->

<!-- 声明所有子模块 -->
<modules>
    <module>common</module>
    <module>api</module>
    <module>service-user</module>
    <module>service-order</module>
    <module>gateway</module>
</modules>

<!-- 统一管理依赖版本（不引入，只声明版本）-->
<dependencyManagement>
    <dependencies>
        <!-- Spring Boot BOM -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-dependencies</artifactId>
            <version>3.2.0</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
        <!-- 内部模块版本也在这里统一管理 -->
        <dependency>
            <groupId>com.example</groupId>
            <artifactId>common</artifactId>
            <version>${project.version}</version>
        </dependency>
    </dependencies>
</dependencyManagement>

<!-- 所有子模块都需要的公共依赖 -->
<dependencies>
    <dependency>
        <groupId>org.projectlombok</groupId>
        <artifactId>lombok</artifactId>
    </dependency>
</dependencies>

<!-- 统一属性 -->
<properties>
    <java.version>17</java.version>
    <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
</properties>
```

---

## Q3. 子模块如何引用其他子模块？

```xml
<!-- service-order/pom.xml -->
<parent>
    <groupId>com.example</groupId>
    <artifactId>my-platform</artifactId>
    <version>1.0.0-SNAPSHOT</version>
    <relativePath>../pom.xml</relativePath>
</parent>

<artifactId>service-order</artifactId>

<dependencies>
    <!-- 引用兄弟模块（版本由父 POM 的 dependencyManagement 管理）-->
    <dependency>
        <groupId>com.example</groupId>
        <artifactId>common</artifactId>
        <!-- 不写 version，继承父 POM 中声明的版本 -->
    </dependency>
    <dependency>
        <groupId>com.example</groupId>
        <artifactId>api</artifactId>
    </dependency>
</dependencies>
```

**注意**：引用兄弟模块时，被引用的模块必须先 install 到本地仓库，否则找不到。多模块一起构建时 Maven 会自动按依赖顺序构建。

---

## Q4. 多模块项目中的构建命令

```bash
# 在父模块目录执行
mvn clean install                   # 构建所有子模块
mvn clean install -DskipTests      # 跳过测试

# 只构建指定子模块
mvn clean install -pl service-user    # -pl 指定模块（相对路径或artifactId）
mvn clean install -pl service-user,service-order  # 多个模块

# 构建指定模块及其依赖的模块
mvn clean install -pl service-order -am    # -am（also-make）

# 构建指定模块及依赖该模块的模块
mvn clean install -pl common -amd          # -amd（also-make-dependents）

# 跳过某个模块
mvn clean install -pl !gateway            # 排除 gateway 模块

# 从某个子模块目录执行（只构建该模块）
cd service-user
mvn clean install
```

---

## Q5. 多模块项目版本管理：如何统一升级版本？

```bash
# 使用 versions-maven-plugin 统一升级
# 在父模块目录执行

# 设置新版本
mvn versions:set -DnewVersion=2.0.0-SNAPSHOT

# 确认更改（生成 .versionsBackup 文件）
mvn versions:commit    # 确认，删除备份文件
# 或
mvn versions:revert    # 撤销，还原备份文件

# 一行命令（不生成备份）
mvn versions:set -DnewVersion=2.0.0-SNAPSHOT -DgenerateBackupPoms=false

# 检查可升级的依赖
mvn versions:display-dependency-updates

# 升级所有依赖到最新版本（慎用生产环境）
mvn versions:use-latest-releases
```

> **追问：大型多模块项目构建慢怎么优化？**
> ①**并行构建**：`mvn clean install -T 4`（4线程）或 `-T 1C`（每核1线程），无依赖关系的模块并行构建；②**增量构建（Gradle 特性，Maven 需要 Takari 插件）**；③**只构建变更模块**：结合 git diff 找到改动的模块，只 `-pl` 构建改动模块及其下游；④**分层构建**：stable 的 common 模块发版后只作为依赖，不每次重新构建；⑤**Daemon 模式**：Maven 3.x 以上自动启用 daemon，减少 JVM 启动开销。
