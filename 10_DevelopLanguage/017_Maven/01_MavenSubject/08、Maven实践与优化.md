# 08、Maven 实践与优化

## Q1. 如何加速 Maven 构建？

```bash
# ==================== 基础加速 ====================

# 1. 并行构建（最有效）
mvn clean install -T 4              # 4个线程
mvn clean install -T 1C             # 每个CPU核1个线程

# 2. 跳过测试（权衡速度和质量）
mvn clean install -DskipTests       # 跳过测试执行（仍编译测试代码）
mvn clean install -Dmaven.test.skip=true  # 不编译也不执行测试

# 3. 离线模式（依赖都已下载时最快）
mvn clean install -o               # offline mode，不访问远程仓库

# 4. 配置国内镜像（网络加速，见仓库章节）

# ==================== 增量编译 ====================
# Maven 默认没有增量编译，可以考虑用 Takari 插件
# 或者只构建有变更的模块（配合 git diff 脚本）

# ==================== 提升编译性能 ====================
# 增大 Maven JVM 内存
export MAVEN_OPTS="-Xms512m -Xmx2g"
# 或使用 .mvn/jvm.config
# 内容：-Xms512m -Xmx2g

# ==================== 减少不必要的操作 ====================
# 按需运行，不总是 clean
mvn install                        # 无 clean，利用上次编译缓存
```

---

## Q2. 如何排查 Maven 依赖冲突？

```bash
# 1. 查看完整依赖树
mvn dependency:tree
mvn dependency:tree -Dverbose     # 显示被排除的依赖（标记omitted for conflict）

# 2. 只查找特定 artifactId
mvn dependency:tree -Dincludes=org.slf4j:slf4j-api
mvn dependency:tree -Dincludes=log4j:*

# 3. 分析依赖使用情况
mvn dependency:analyze
# 输出：
# Used undeclared dependencies（用了但没声明，依赖传递来的）→ 需要显式声明
# Unused declared dependencies（声明了但没用）→ 考虑删除

# 4. 列出所有依赖（包含传递性依赖）
mvn dependency:list | sort

# ==================== 解决冲突 ====================
# 方案1：在 dependencyManagement 中锁定版本
# 方案2：exclusion 排除冲突的传递性依赖，再显式声明正确版本
<dependency>
    <groupId>org.example</groupId>
    <artifactId>some-lib</artifactId>
    <exclusions>
        <exclusion>
            <groupId>org.slf4j</groupId>
            <artifactId>slf4j-api</artifactId>
        </exclusion>
    </exclusions>
</dependency>
<dependency>
    <groupId>org.slf4j</groupId>
    <artifactId>slf4j-api</artifactId>
    <version>2.0.9</version>
</dependency>
```

---

## Q3. Maven 构建失败时如何排查？

```bash
# 开启详细日志
mvn clean install -X               # DEBUG 级别日志（非常详细）
mvn clean install -e               # 打印异常堆栈（比 -X 少一点）

# 常见错误排查

# 1. "Could not find artifact xxx" → 依赖找不到
# - 检查坐标是否正确（groupId/artifactId/version）
# - 检查仓库配置（settings.xml 镜像、pom.xml repositories）
# - 尝试 mvn clean install -U 强制更新
# - 手动删除 ~/.m2/repository/xxx 损坏的缓存文件（.lastUpdated 文件）

# 2. "Compilation failure" → 编译错误
# - 查看具体的编译错误信息
# - 注意 Java 版本配置是否正确

# 3. "Tests failed" → 测试失败
# - 查看 target/surefire-reports/ 下的测试报告
# - 用 -Dtest=FailedTest 单独运行失败的测试排查

# 4. "BUILD FAILURE without error message" → 通常是插件配置错误
# - 加 -e 参数查看详细堆栈

# 清理损坏的本地缓存
find ~/.m2/repository -name "*.lastUpdated" -delete
find ~/.m2/repository -name "_remote.repositories" -delete
```

---

## Q4. Maven 和 Gradle 的区别？

| 维度 | Maven | Gradle |
|------|-------|--------|
| **配置语言** | XML（冗长但直观）| Groovy/Kotlin DSL（简洁但需要学习）|
| **构建速度** | 较慢，全量构建 | 快，增量构建+构建缓存 |
| **灵活性** | 约定死板，定制复杂 | 高度灵活，任意逻辑 |
| **学习曲线** | 较平缓，到处有示例 | 较陡峭，DSL 需要学习 |
| **生态成熟度** | 极成熟，Java 传统首选 | 快速成熟，Android 官方 |
| **IDE 支持** | 完美 | 完美（IntelliJ 深度集成）|
| **性能（大项目）** | 差距明显 | 快 2-5 倍（增量+缓存）|
| **Spring Boot 官方** | 优先展示 | 同等支持 |

**选择建议**：
- 新建 Java 项目且团队有 Gradle 经验 → **Gradle**（构建更快，配置更简洁）
- 存量 Maven 项目维护 → **继续 Maven**（迁移成本高于收益）
- Android 项目 → **Gradle**（官方指定）
- 大型多模块项目关注构建速度 → **Gradle**（增量构建优势明显）
- 团队技术栈以 Java 为主，不想学新工具 → **Maven**

---

## Q5. Maven 最佳实践有哪些？

**依赖管理最佳实践**：
```xml
<!-- 1. 使用父 POM 统一管理版本 -->
<!-- 2. 优先用 BOM 而不是逐个指定版本 -->
<!-- 3. 定期运行 dependency:analyze，清理未使用依赖 -->
<!-- 4. 生产依赖不用 SNAPSHOT 版本 -->
<!-- 5. 明确声明 scope，避免无谓的 compile scope -->
```

**构建配置最佳实践**：
```xml
<!-- 1. 明确指定插件版本（不依赖 Maven 默认版本）-->
<plugin>
    <artifactId>maven-compiler-plugin</artifactId>
    <version>3.13.0</version>  <!-- 明确版本 -->
</plugin>

<!-- 2. 指定编码，避免跨平台乱码 -->
<properties>
    <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
    <project.reporting.outputEncoding>UTF-8</project.reporting.outputEncoding>
</properties>

<!-- 3. 配置 Maven Wrapper，统一团队 Maven 版本 -->
<!-- 4. 将 settings.xml 中的密码加密存储 -->
<!-- mvn --encrypt-master-password mypassword -->
<!-- mvn --encrypt-password myRepoPassword -->
```

**项目结构最佳实践**：
- 多模块项目中父 POM 只做聚合和版本管理，不写业务代码
- 公共依赖放父 POM 的 `dependencies`，只在子模块用的依赖放子模块 pom
- 测试代码和生产代码分开（`src/test/java` vs `src/main/java`）

---

## Q6. 如何与 Jenkins 集成进行 CI/CD？

```groovy
// Jenkinsfile（Declarative Pipeline）
pipeline {
    agent any
    
    tools {
        maven 'Maven-3.9'      // Jenkins 中配置的 Maven
        jdk 'JDK-17'
    }
    
    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }
        
        stage('Build') {
            steps {
                sh 'mvn clean compile'
            }
        }
        
        stage('Test') {
            steps {
                sh 'mvn test'
            }
            post {
                always {
                    // 收集测试报告
                    junit '**/target/surefire-reports/*.xml'
                    // JaCoCo 覆盖率报告
                    jacoco(execPattern: '**/target/jacoco.exec')
                }
            }
        }
        
        stage('Package') {
            steps {
                sh 'mvn clean package -DskipTests'
                archiveArtifacts artifacts: '**/target/*.jar', fingerprint: true
            }
        }
        
        stage('Deploy to Nexus') {
            when { branch 'main' }
            steps {
                sh 'mvn deploy -DskipTests -s /var/jenkins/settings.xml'
            }
        }
    }
    
    post {
        failure {
            emailext(
                to: 'team@company.com',
                subject: "Build Failed: ${env.JOB_NAME}",
                body: "Check ${env.BUILD_URL}"
            )
        }
    }
}
```

---

## Q7. settings.xml 和 pom.xml 的区别？

| 维度 | settings.xml | pom.xml |
|------|-------------|---------|
| **位置** | `~/.m2/settings.xml`（用户级）或 `${MAVEN_HOME}/conf/settings.xml`（全局）| 项目根目录 |
| **版本控制** | ❌ 不提交（含密码、私有仓库地址）| ✅ 必须提交 |
| **作用范围** | 该机器上所有 Maven 项目 | 当前项目 |
| **配置内容** | 认证信息、镜像、代理、本地仓库路径 | 依赖、插件、构建配置、profiles |
| **典型用途** | 配置私服账号、设置国内镜像 | 声明项目依赖、打包方式 |

**经验**：敏感信息（私服密码）放 settings.xml 并加密，不要放 pom.xml（会被提交到 Git）。
