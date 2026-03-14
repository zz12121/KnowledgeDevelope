###### 1. JVM 在 Docker 容器中的内存问题？

[[../../../20_JavaKnowledge/04_JVM/12、JVM与容器化（Docker-K8s）#一、容器化带来的 JVM 问题|📖]]
这是一个经典的"坑"。JDK 8 Update 191 之前，JVM **无法感知 Docker 容器的内存限制**，它直接读取的是宿主机的总内存。

问题场景：容器限制内存为 2GB，但宿主机有 64GB 内存，JVM 看到的是 64GB，会把堆大小默认设为 64GB / 4 = 16GB，远超容器限制，导致容器 OOM Killed 被杀。

**解决方案**：
1. **JDK 8u191+（推荐）**：开启 `-XX:+UseContainerSupport`（JDK 8u191+ 默认开启），JVM 会正确读取 cgroups 的内存限制
2. **明确指定堆大小**：`-Xms512m -Xmx1g`，不依赖 JVM 自动计算，简单粗暴但有效
3. **JDK 11+**：默认支持容器感知，不需要额外配置

###### 2. 如何在容器中正确设置 JVM 参数？

[[../../../20_JavaKnowledge/04_JVM/12、JVM与容器化（Docker-K8s）#二、容器内堆大小配置|📖]]
容器环境下推荐的 JVM 参数设置：

```bash
# 内存设置（使用容器感知）
-XX:+UseContainerSupport
-XX:MaxRAMPercentage=75.0     # 堆最大使用容器内存的75%，给 OS、GC 元数据、直接内存留空间
-XX:InitialRAMPercentage=50.0 # 堆初始大小

# 或者直接指定（更明确）
-Xms512m -Xmx1536m            # 容器2GB，留512MB给非堆内存

# GC 设置
-XX:+UseG1GC
-XX:MaxGCPauseMillis=200

# 调试便利
-XX:+HeapDumpOnOutOfMemoryError
-XX:HeapDumpPath=/logs/heap.hprof
-Xlog:gc*:file=/logs/gc.log:time,uptime:filecount=5,filesize=20m
```

不要在容器里把堆设得和容器内存一样大，至少要留出：
- 元空间（通常 100~500MB）
- 线程栈（线程数 × Xss）
- 直接内存（NIO 使用的堆外内存）
- JVM 内部结构

经验原则：**堆大小 ≤ 容器内存 × 75%**。

###### 3. 什么是 -XX:+UseContainerSupport？

[[../../../20_JavaKnowledge/04_JVM/12、JVM与容器化（Docker-K8s）#三、CPU 资源配置|📖]]
`-XX:+UseContainerSupport` 是让 JVM 正确识别容器（Docker / Kubernetes）内存和 CPU 限制的参数：

**JDK 8u191、JDK 10+ 默认开启**。开启后 JVM 会读取 cgroups 的限制信息而不是宿主机的总资源：
- 内存：读取 `/sys/fs/cgroup/memory/memory.limit_in_bytes` 获取容器内存限制
- CPU：读取 cgroups 的 CPU 配额来计算可用 CPU 核心数（影响 GC 线程数、JIT 线程数等）

还可以配合以下参数精细控制：
- `-XX:MaxRAMPercentage=75.0`：堆使用容器总内存的百分比（推荐）
- `-XX:MinRAMPercentage=50.0`：容器内存 < 200MB 时使用的百分比
- `-XX:InitialRAMPercentage=50.0`：初始堆使用容器总内存的百分比

###### 4. 容器环境下的 CPU 限制问题？

[[../../../20_JavaKnowledge/04_JVM/12、JVM与容器化（Docker-K8s）#四、K8s 探针与 JVM 启动|📖]]
**问题**：和内存类似，老版本 JDK 也无法感知 cgroups 的 CPU 限制，会读取宿主机的 CPU 核心数。如果容器只限制了0.5个 CPU，但宿主机有32核，JVM 会启动32个 GC 线程，32个 JIT 编译线程，CPU 竞争激烈，反而比单线程慢。

**解决方案**：
- 使用 JDK 8u191+ 并开启 `UseContainerSupport`，JVM 会根据 cgroups 配额计算逻辑 CPU 数
- 或者手动指定：`-XX:ActiveProcessorCount=2`（强制告诉 JVM 有2个可用 CPU）
- 对于 G1：`-XX:ParallelGCThreads=2 -XX:ConcGCThreads=1`（GC 线程数不超过可用 CPU）

###### 5. 如何优化容器中的 JVM 启动时间？

[[../../../20_JavaKnowledge/04_JVM/12、JVM与容器化（Docker-K8s）#五、容器化常见问题与解决|📖]]
容器频繁启停（微服务、Serverless、弹性伸缩）时，JVM 启动慢是个痛点（特别是 Spring Boot 应用动辄几秒到十几秒）：

**方案一：CDS（Class Data Sharing）**：JVM 把加载、链接、初始化后的类数据保存成归档文件，下次启动直接映射到内存，减少重复工作。JDK 8 就支持，JDK 12+ 的 AppCDS 更好用（见下题）。

**方案二：减少类加载**：启动时按需加载（懒加载），不要一次性加载所有 bean 和配置。Spring Boot 2.2+ 支持 `spring.main.lazy-initialization=true`。

**方案三：调整 JIT 策略**：容器启动阶段不需要激进优化，可以用 `-XX:TieredStopAtLevel=1` 只用 C1 编译，启动更快（但长期运行性能差，适合短生命周期容器）。

**方案四：GraalVM Native Image**：提前把应用编译成本地可执行文件，毫秒级启动，但有一些限制（见第9题）。

###### 6. 什么是 CDS（Class Data Sharing）？

[[../../../20_JavaKnowledge/04_JVM/12、JVM与容器化（Docker-K8s）#六、JVM 容器化最佳实践|📖]]
CDS（Class Data Sharing，类数据共享）是 JVM 的一种优化机制：**把一组类（通常是 JDK 核心类）的处理结果（共享归档文件）保存下来，下次 JVM 启动时多个进程直接共享这份数据**，跳过重复的加载、链接步骤。

**好处**：
- 减少启动时间（跳过类加载步骤）
- 减少内存占用（多个 JVM 实例共享同一份归档文件，用内存映射实现）

**使用步骤**（JDK 8）：
```bash
# 1. 生成类列表
java -Xshare:dump

# 2. 下次启动使用共享归档
java -Xshare:on -jar myapp.jar
```

JDK 9+ 的 AppCDS 扩展了 CDS，可以包含应用类而不仅限于 JDK 核心类。

###### 7. 什么是 AppCDS？如何使用？

[[../../../20_JavaKnowledge/04_JVM/12、JVM与容器化（Docker-K8s）#一、容器化带来的 JVM 问题|📖]]
AppCDS（Application Class Data Sharing）是 JDK 10 正式引入（JDK 8u202 商业版有）的增强版 CDS，支持把**应用自身的类**也加入共享归档，进一步减少启动时间。

**使用步骤**：
```bash
# 第一步：记录类加载列表（运行一次应用，收集加载了哪些类）
java -XX:DumpLoadedClassList=classes.lst -jar myapp.jar

# 第二步：生成共享归档文件
java -Xshare:dump -XX:SharedClassListFile=classes.lst \
     -XX:SharedArchiveFile=app-cds.jsa -jar myapp.jar

# 第三步：使用归档启动（后续每次启动）
java -Xshare:on -XX:SharedArchiveFile=app-cds.jsa -jar myapp.jar
```

对于 Spring Boot 这类框架，AppCDS 通常可以减少 20~40% 的启动时间，是容器化场景下提升启动速度性价比最高的方法之一。

JDK 13 起支持**动态 CDS 归档**（`-XX:ArchiveClassesAtExit`），应用退出时自动生成归档，更简单。

###### 8. 什么是 AOT（Ahead-Of-Time）编译？

[[../../../20_JavaKnowledge/04_JVM/12、JVM与容器化（Docker-K8s）#一、容器化带来的 JVM 问题|📖]]
AOT（Ahead-Of-Time Compilation，提前编译）是相对 JIT（Just-In-Time，即时编译）的一种编译策略：**在程序运行之前就把代码编译成本地机器码**，而不是运行时动态编译。

**优势**：
- 没有 JIT 的预热时间，程序**启动就是最优性能**
- 内存占用更小（不需要存储字节码和 JIT 编译后的多份代码）

**劣势**：
- 失去了 JIT 根据运行时 profiling 数据做动态优化的能力（JIT 能做自适应内联、激进去虚化等 AOT 做不到的优化）
- Java 的动态特性（反射、动态代理、字节码操纵）支持有限，需要提前声明配置

JDK 9 引入的 `jaotc` 工具是 HotSpot 的 AOT 实现（已在 JDK 17 移除），GraalVM Native Image 是目前最成熟的 Java AOT 方案。

###### 9. GraalVM 是什么？与传统 JVM 的区别？

[[../../../20_JavaKnowledge/04_JVM/12、JVM与容器化（Docker-K8s）#一、容器化带来的 JVM 问题|📖]]
GraalVM 是 Oracle 开发的**多语言虚拟机**，不仅仅是 Java，还支持 JavaScript、Python、Ruby、LLVM 语言等在同一个运行时内运行，甚至可以互调。

**GraalVM 的主要组件**：
- **Graal 编译器**：高质量的 JIT 编译器，替换 HotSpot 的 C2，用 Java 写的 Java 编译器，比 C2 做更激进的优化
- **Truffle 框架**：为其他语言实现解释器的框架，让 Python/JS 等语言跑在 GraalVM 上能享受 JIT 优化
- **Native Image**：AOT 编译，把 Java 应用编译成本地可执行文件（见下题）

**与传统 HotSpot JVM 的区别**：

传统 JVM 启动慢（需要 JIT 预热）、内存占用大（需要加载 JVM 本身）、支持所有 Java 动态特性。GraalVM 可选用 JIT 模式（和 HotSpot 类似但编译器更强）或 Native Image AOT 模式（启动极快、内存极小，但动态特性受限）。

###### 10. Native Image 的优势和局限性？

[[../../../20_JavaKnowledge/04_JVM/12、JVM与容器化（Docker-K8s）#一、容器化带来的 JVM 问题|📖]]
Native Image 是 GraalVM 最有价值的特性，把 Java 应用提前编译成**平台相关的原生可执行文件**（无需 JVM 运行时）。

**优势**：
- **启动极快**：毫秒级启动（传统 Spring Boot 应用通常需要几秒），Serverless、命令行工具场景完美
- **内存极小**：没有 JVM 运行时，只包含程序实际用到的代码，内存占用大幅减少（Spring Boot 应用从几百 MB 降到几十 MB）
- **无 JIT 预热**：从第一个请求开始就是最优性能
- **安全性**：攻击面更小，没有字节码，没有 JIT 编译器

**局限性**：
- **动态特性受限**：反射、动态代理、JNI、序列化等动态特性需要**提前声明配置**（reflection-config.json、proxy-config.json 等），否则编译时静态分析找不到就不会包含进去
- **编译慢、资源消耗大**：Native Image 编译需要几分钟，需要大量内存（4~8GB 以上），不适合频繁构建
- **运行时优化不如 JIT**：没有 profile 驱动的动态优化，长期运行的峰值吞吐量可能不如 JIT
- **框架支持**：需要框架提供 Native Image 支持（Spring Boot 3.x 官方支持 GraalVM Native Image，Quarkus 和 Micronaut 也是为此而生的）

**适用场景**：Serverless 函数（AWS Lambda 等）、命令行工具（需要快速启动）、微服务的首批请求敏感场景、云原生边缘计算。
