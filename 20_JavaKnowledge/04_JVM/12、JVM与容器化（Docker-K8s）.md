# JVM 与容器化（Docker/K8s）

> **核心关键词**：容器资源限制、cgroup、JVM 感知容器、堆大小自动计算、MaxRAMPercentage、就绪探针、JVM 调优

---

## 一、容器化带来的 JVM 问题

### 1.1 JVM 不感知容器的问题（JDK 8u191 之前）

```
问题根源：JVM 默认读取物理机的 CPU 和内存信息
容器通过 cgroup 限制资源，但旧版 JVM 看到的是宿主机资源

假设场景：
  宿主机：64 核 CPU，128GB 内存
  容器限制：2 核 CPU，4GB 内存

旧版 JVM 的问题：
  1. 堆大小默认 = 物理内存 / 4 → 128GB / 4 = 32GB！！！
     → 容器只有 4GB，立刻 OOM Killed
  2. GC 并行线程数 = CPU 核数 → 64 个 GC 线程
     → 容器只有 2 核，严重争用 CPU
  3. Fork/Join 线程池大小 = CPU 核数 → 64 个线程
     → 容器资源超用
```

### 1.2 JDK 8u191+ / JDK 10+ 的修复

```bash
# JDK 8u191+、JDK 10+ 开始支持容器感知
# 关键 JVM 参数：

# 开启容器支持（JDK 8u191~8u249 需显式开启，JDK 10+ 默认开启）
-XX:+UseContainerSupport

# 验证容器感知是否生效
java -XX:+PrintFlagsFinal -version | grep -i 'usecontainer\|maxram'
```

---

## 二、容器内堆大小配置

### 2.1 推荐方式：MaxRAMPercentage

```bash
# 方式1：按容器内存百分比设置堆（推荐）
-XX:MaxRAMPercentage=75.0    # 最大堆 = 容器内存 × 75%
-XX:InitialRAMPercentage=50.0 # 初始堆 = 容器内存 × 50%
-XX:MinRAMPercentage=50.0    # 容器内存 ≤ 200MB 时，最大堆 = 容器内存 × 50%

# 示例：容器 4GB 内存
# MaxRAMPercentage=75.0 → 最大堆 = 3GB

# 为什么不设 100%？
# JVM 除堆外还需要：
#   元空间（Metaspace）：几百MB
#   直接内存（DirectMemory）：几百MB
#   JVM 内部结构、线程栈：几百MB
# 预留 20-25% 给堆外内存，通常建议 60-75%

# 方式2：固定大小（传统方式，需根据容器规格手动设置）
-Xms2g -Xmx3g   # 容器 4GB 内存时

# 验证当前 JVM 看到的最大堆
java -XX:+PrintFlagsFinal -version 2>&1 | grep MaxHeapSize
# 或运行时查看
jcmd <PID> VM.flags | grep HeapSize
```

### 2.2 Dockerfile 最佳实践

```dockerfile
FROM eclipse-temurin:21-jre

ENV JAVA_OPTS="-XX:+UseContainerSupport \
               -XX:MaxRAMPercentage=75.0 \
               -XX:InitialRAMPercentage=50.0 \
               -XX:+UseG1GC \
               -XX:MaxGCPauseMillis=200 \
               -XX:+HeapDumpOnOutOfMemoryError \
               -XX:HeapDumpPath=/tmp/heapdump.hprof \
               -Xlog:gc*:file=/var/log/app/gc.log:time,uptime:filecount=5,filesize=20m"

COPY target/app.jar /app/app.jar

ENTRYPOINT ["sh", "-c", "java $JAVA_OPTS -jar /app/app.jar"]
```

---

## 三、CPU 资源配置

### 3.1 JVM 的 CPU 感知

```bash
# JDK 10+ 开启容器支持后，JVM 会读取 cgroup 的 CPU 配额
# 自动计算合适的线程数

# 容器 CPU 核数计算：
# cfs_quota_us / cfs_period_us = 可用 CPU 数
# 如：quota=200000, period=100000 → 2 核

# JVM 自动调整的参数：
#   -XX:ActiveProcessorCount  （被容器支持覆盖）
#   GC 并行线程数（-XX:ParallelGCThreads）
#   JIT 编译线程数
#   ForkJoinPool 并行度

# 手动覆盖（在自动计算不准确时使用）
-XX:ActiveProcessorCount=2   # 告诉 JVM 可用 2 个 CPU
```

### 3.2 K8s 资源配置与 JVM

```yaml
# Kubernetes Deployment 示例
spec:
  containers:
    - name: app
      image: myapp:1.0
      resources:
        requests:
          memory: "2Gi"
          cpu: "1"
        limits:
          memory: "4Gi"    # 容器内存上限，JVM 感知此值
          cpu: "2"         # CPU 上限，JVM 据此调整线程数
      env:
        - name: JAVA_OPTS
          value: >-
            -XX:+UseContainerSupport
            -XX:MaxRAMPercentage=75.0
            -XX:+UseG1GC
            -XX:MaxGCPauseMillis=200
```

---

## 四、K8s 探针与 JVM 启动

### 4.1 JVM 预热问题

```
JVM 启动过程：
  1. JVM 初始化
  2. 类加载（Spring Boot 加载大量类）
  3. 容器启动完成，K8s readinessProbe 开始检测
  4. 接收流量，JIT 开始编译热点代码（CPU 飙高属正常）
  5. 预热完成（分钟级），性能稳定

问题：
  K8s 探针过早认为服务 Ready，流量涌入但 JVM 还在预热
  → 响应慢，甚至触发熔断
```

```yaml
# K8s 探针配置示例
livenessProbe:
  httpGet:
    path: /actuator/health/liveness
    port: 8080
  initialDelaySeconds: 60   # JVM 启动 + Spring Boot 初始化时间
  periodSeconds: 10
  failureThreshold: 3

readinessProbe:
  httpGet:
    path: /actuator/health/readiness
    port: 8080
  initialDelaySeconds: 30   # 等待应用初始化
  periodSeconds: 5
  successThreshold: 1

# Spring Boot 3.x 支持 Graceful Shutdown
# application.yaml
server:
  shutdown: graceful    # K8s Pod 删除时优雅停机
spring:
  lifecycle:
    timeout-per-shutdown-phase: 30s
```

### 4.2 JVM 预热方案

```java
// 方案1：Startup Warmup（Spring Boot Actuator）
// 通过 ApplicationRunner 预加载关键代码路径

@Component
public class JvmWarmupRunner implements ApplicationRunner {
    @Override
    public void run(ApplicationArguments args) {
        // 调用核心接口几次，触发 JIT 编译
        for (int i = 0; i < 100; i++) {
            try {
                warmupCriticalPaths();
            } catch (Exception ignored) {}
        }
        log.info("JVM warmup complete");
    }
}

// 方案2：CRaC（Coordinated Restore at Checkpoint）
// JDK 21+ 特性：在预热后对 JVM 做快照，之后启动直接恢复快照
// 启动时间从秒级缩短到毫秒级（Spring Boot 3.2+ 支持）
```

---

## 五、容器化常见问题与解决

### 5.1 OOM Killed

```bash
# 现象：容器被 K8s 杀掉（exit code 137 = OOM Killed）
# K8s events：OOMKilled

# 原因排查：
kubectl describe pod <pod-name>  # 查看 OOM 记录

# 常见原因：
# 1. 堆外内存（Metaspace + DirectMemory + 线程栈）超出限制
#    解决：减少 MaxRAMPercentage，或增加内存 limits

# 2. 默认 JVM 未感知容器限制（旧版 JDK）
#    解决：升级 JDK 或显式加 -XX:+UseContainerSupport

# 3. 有内存泄漏
#    解决：dump 分析（但容器重启后数据丢失，需提前配置 hostPath volume 保存 dump）

# 建议：always set memory limits = requests（避免节点内存争用）
# 堆 + 堆外内存经验公式：
# container_memory ≥ Xmx × 1.25 + 512MB（元空间 + 直接内存 + 线程栈）
```

### 5.2 内存溢出时自动 dump

```yaml
# 挂载持久化存储保存 heap dump
spec:
  containers:
    - name: app
      env:
        - name: JAVA_OPTS
          value: >-
            -XX:+HeapDumpOnOutOfMemoryError
            -XX:HeapDumpPath=/dumps/heapdump.hprof
      volumeMounts:
        - name: dumps
          mountPath: /dumps
  volumes:
    - name: dumps
      persistentVolumeClaim:
        claimName: app-dumps-pvc
```

### 5.3 GC 日志持久化

```yaml
# 将 GC 日志写入 volume，配合 filebeat 收集
- name: JAVA_OPTS
  value: >-
    -Xlog:gc*:file=/var/log/gc/gc.log:time,uptime:filecount=10,filesize=20m
volumeMounts:
  - name: gc-logs
    mountPath: /var/log/gc
```

---

## 六、JVM 容器化最佳实践

```
1. 使用 JDK 11+ 镜像（ContainerSupport 默认开启）
   推荐：eclipse-temurin（Adoptium）、amazoncorretto

2. 设置 MaxRAMPercentage 代替固定 -Xmx
   小内存容器（< 1GB）：50-60%
   普通容器（1-8GB）：70-75%
   大内存容器（8GB+）：75-80%

3. 合理设置 requests 和 limits
   memory: limits = requests（避免 OOM 时调度器将 Pod 驱逐）
   cpu: requests < limits（允许 CPU burst）

4. 配置优雅停机
   server.shutdown=graceful
   terminationGracePeriodSeconds: 60（K8s 等待时间）

5. 必须配置 OOM Heap Dump
   -XX:+HeapDumpOnOutOfMemoryError + 持久化 Volume

6. 监控 JVM 指标
   Micrometer + Prometheus + Grafana（JVM 内存/GC/线程）

7. 注意 JVM 启动时间
   Spring Boot 重量级应用：30-120 秒
   配置合理的 initialDelaySeconds
   JDK 21 CRaC / GraalVM Native Image 可大幅缩短启动时间
```

---

## 七、面试要点速查

| 问题 | 要点 |
|------|------|
| 为什么容器内 JVM 会 OOM Killed | 旧版 JVM 读物理机内存设堆大小，超出容器 limits → OOM Killed |
| UseContainerSupport 的作用 | 让 JVM 读 cgroup 中的内存/CPU 限制，而非物理机信息 |
| MaxRAMPercentage 推荐值 | 60-75%，剩余留给 Metaspace、DirectMemory、线程栈 |
| K8s Pod OOM 如何排查 | `kubectl describe pod` 查看 OOM 事件；分析 heap dump（需提前配置持久化）|
| 容器化的 JVM 线程数如何计算 | JDK 10+ 自动读取 cgroup CPU 配额，或用 ActiveProcessorCount 手动指定 |
| JVM 在容器内的启动预热 | Spring Boot 启动慢，需合理配置 readinessProbe 的 initialDelaySeconds |


---

**相关面试题** → [[../../10_Developlanguage/001_java/04_JavaJVMSubject/09、JVM 与容器化|09、JVM 与容器化]]
