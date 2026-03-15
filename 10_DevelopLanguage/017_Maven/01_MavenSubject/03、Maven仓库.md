# 03、Maven 仓库

## Q1. Maven 仓库有哪些类型？

Maven 仓库分三类，查找依赖时按顺序从左到右：

```
本地仓库（Local） → 私服（Nexus/Artifactory） → 中央仓库（Central）
```

**本地仓库**：`~/.m2/repository/`，下载过的 jar 会缓存到这里，下次直接用本地缓存。

**远程仓库**：分两种：
- **中央仓库（Maven Central）**：`https://repo1.maven.org/maven2/`，包含绝大多数开源库，是 Maven 内置的默认远程仓库
- **私服（Private Repository）**：公司自己搭建的仓库（Nexus OSS 最主流），介于本地和中央之间

---

## Q2. 私服的作用是什么？为什么需要私服？

私服（如 Sonatype Nexus）在企业中几乎是标配：

**核心价值**：
1. **加速构建**：依赖从公司内网私服下载，比直接从 Maven Central 快 10 倍以上
2. **内部包发布**：公司内部的 SDK/公共库发布到私服，各项目共享复用
3. **隔离外网**：生产构建环境不需要访问公网，安全性更强
4. **稳定性**：不依赖 Maven Central 的可用性，中央仓库宕机不影响构建
5. **版本控制**：统一管理所有依赖版本，防止开发人员随意引入不安全的依赖

**Nexus 仓库类型**：
- `hosted`：存放自研包（releases/snapshots）
- `proxy`：代理外部仓库（Maven Central/阿里云等）
- `group`：聚合多个仓库，对外提供统一的访问地址

---

## Q3. 如何配置私服（settings.xml）？

```xml
<!-- ~/.m2/settings.xml -->
<settings>
    <!-- 配置私服的认证信息 -->
    <servers>
        <server>
            <id>nexus-releases</id>   <!-- 与 distributionManagement 中的 id 对应 -->
            <username>admin</username>
            <password>admin123</password>
        </server>
        <server>
            <id>nexus-snapshots</id>
            <username>admin</username>
            <password>admin123</password>
        </server>
    </servers>

    <!-- 配置所有项目都通过私服下载依赖（镜像配置）-->
    <mirrors>
        <mirror>
            <id>nexus-mirror</id>
            <name>Nexus Mirror</name>
            <url>http://nexus.company.com/repository/maven-public/</url>
            <mirrorOf>*</mirrorOf>   <!-- * 表示镜像所有远程仓库 -->
        </mirror>
    </mirrors>

    <!-- 如果只想加速 central，不想镜像其他仓库 -->
    <!-- <mirrorOf>central</mirrorOf> -->
</settings>
```

---

## Q4. 如何将项目发布到私服？

在项目的 `pom.xml` 中配置 `distributionManagement`：

```xml
<distributionManagement>
    <!-- 发布版本（RELEASE）仓库 -->
    <repository>
        <id>nexus-releases</id>   <!-- 与 settings.xml 中 server id 对应 -->
        <url>http://nexus.company.com/repository/maven-releases/</url>
    </repository>
    <!-- 快照版本（SNAPSHOT）仓库 -->
    <snapshotRepository>
        <id>nexus-snapshots</id>
        <url>http://nexus.company.com/repository/maven-snapshots/</url>
    </snapshotRepository>
</distributionManagement>
```

```bash
# 发布到私服
mvn deploy                        # 发布当前项目
mvn deploy -DskipTests           # 跳过测试发布

# 多模块项目发布
mvn clean deploy -pl module-a    # 只发布 module-a
```

---

## Q5. Maven 下载依赖的完整流程是什么？

```
1. 在本地仓库查找
   找到 → 直接使用
   没找到 ↓

2. 查找远程仓库列表（settings.xml 中的 mirrors + pom.xml 中的 repositories）

3. 按优先级依次请求：
   ├─ 私服 proxy 仓库（代理中央仓库）
   │    ├─ 私服缓存有 → 返回
   │    └─ 没有 → 私服去中央仓库拉取 → 缓存到私服 → 返回给客户端
   └─ 如果没配私服，直接访问 Maven Central

4. 下载到本地仓库缓存（~/.m2/repository/）

5. 后续构建直接用本地缓存
```

**SNAPSHOT 版本的特殊处理**：
- SNAPSHOT 版本每次构建都会检查远程仓库是否有更新（根据 `updatePolicy` 配置）
- RELEASE 版本一旦下载到本地就不再重新下载（除非手动删除本地缓存）
- 可以用 `-U` 参数强制更新：`mvn clean install -U`

---

## Q6. 配置国内镜像加速（阿里云）

```xml
<!-- ~/.m2/settings.xml -->
<mirrors>
    <mirror>
        <id>aliyun</id>
        <name>阿里云 Maven 镜像</name>
        <url>https://maven.aliyun.com/repository/public</url>
        <mirrorOf>*</mirrorOf>
    </mirror>
</mirrors>
```

或者在 `pom.xml` 中指定（仅对该项目生效）：
```xml
<repositories>
    <repository>
        <id>aliyun</id>
        <url>https://maven.aliyun.com/repository/public</url>
        <snapshots>
            <enabled>false</enabled>
        </snapshots>
    </repository>
</repositories>
```

> **追问：settings.xml 在哪里？优先级怎么样？**
> 有两个位置：全局配置 `${MAVEN_HOME}/conf/settings.xml`，用户配置 `~/.m2/settings.xml`。用户配置优先级更高，相同配置项用户配置覆盖全局配置，不冲突的部分合并。实际工作中一般只改用户配置，不动全局配置，方便版本升级。
