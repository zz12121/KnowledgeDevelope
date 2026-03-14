# JVM 调优实战

> **核心关键词**：GC 日志分析、堆内存配置、GC 选型、OOM 排查、CPU 飙高、内存泄漏、jstat、jmap、Arthas

---

## 一、JVM 调优的目标与原则

```
调优目标（三要素，通常需要取舍）：
  吞吐量（Throughput）：单位时间内完成的工作量
  停顿时间（Latency）：GC 暂停的时长（P99/P999）
  内存占用（Footprint）：堆内存的使用量

调优原则：
  1. 优先通过业务优化（减少对象创建）而非调 JVM 参数
  2. 先解决 Full GC，再优化 Young GC
  3. 不要过早优化，有监控数据支撑再改
  4. 每次只改一个参数，观察效果后再改下一个
  5. 线上调参前，先在预发/压测环境验证
```

---

## 二、基础配置参数

```bash
# 堆大小（建议 Xms = Xmx，避免动态扩容开销）
-Xms4g -Xmx4g

# 新生代大小（或用比例 -XX:NewRatio=2 表示 Old:New=2:1）
-Xmn1g

# 元空间
-XX:MetaspaceSize=256m -XX:MaxMetaspaceSize=512m

# 直接内存
-XX:MaxDirectMemorySize=512m

# GC 选择
-XX:+UseG1GC            # G1（JDK 9+ 默认）
-XX:+UseZGC             # ZGC（低延迟首选）
-XX:+UseConcMarkSweepGC # CMS（JDK 9 废弃，不推荐）
-XX:+UseParallelGC      # Parallel GC（高吞吐量批处理）

# OOM 自动 dump
-XX:+HeapDumpOnOutOfMemoryError
-XX:HeapDumpPath=/tmp/heapdump.hprof

# GC 日志（JDK 9+）
-Xlog:gc*:file=/var/log/app/gc.log:time,uptime,level,tags:filecount=5,filesize=20m

# JDK 8 GC 日志
-verbose:gc -XX:+PrintGCDetails -XX:+PrintGCDateStamps -Xloggc:/var/log/app/gc.log
```

---

## 三、GC 日志分析

### 3.1 读懂 G1 GC 日志

```
[2026-03-14T20:13:00.123+0800] GC(42) Pause Young (Normal) (G1 Evacuation Pause)
[2026-03-14T20:13:00.123+0800] GC(42)   Heap: 2048M->1024M(4096M)
[2026-03-14T20:13:00.156+0800] GC(42) Pause Young (Normal) 2048M->1024M(4096M) 33.2ms

解读：
  Young GC（正常晋升）
  堆：从 2048M → 1024M（堆总大小 4096M）
  停顿时间：33.2ms

关注点：
  - Pause Young 停顿时间是否超过 MaxGCPauseMillis
  - Heap 回收后大小（持续增长说明老年代压力大）
  - To-space exhausted（晋升失败，危险信号！）
  - Humongous Allocation（大对象频繁分配）
```

### 3.2 常用 GC 日志分析工具

```
GCViewer：图形化分析 GC 日志，显示吞吐量/停顿分布
GCEasy（https://gceasy.io）：在线 GC 日志分析，自动给出优化建议
JVM GC Analyzer：IDEA 插件
Grafana + JMX Exporter + Prometheus：生产监控实时大盘
```

---

## 四、常见问题诊断

### 4.1 Full GC 频繁

```bash
# 问题特征：GC 日志中频繁出现 Full GC / Pause Full
# 原因排查：

# 1. 查看老年代占用
jstat -gcold <PID> 1000 10
# 若 OU（老年代已用）持续接近 OC（老年代容量） → 老年代压力大

# 2. 排查内存泄漏
jmap -histo:live <PID> | head -30  # 查看存活对象排名
# 或 OOM 时自动生成的 heap dump
jmap -dump:format=b,file=/tmp/heap.hprof <PID>

# 3. 用 MAT（Memory Analyzer Tool）分析 heap dump
# 打开 heap.hprof → Leak Suspects → 找到内存泄漏嫌疑

# 常见原因：
# - ThreadLocal 未 remove（线程池中泄漏）
# - 静态集合持有大量对象引用
# - 大量 Humongous 对象（ZGC/G1 中）
# - 元空间泄漏（动态生成大量类）
```

### 4.2 CPU 飙高

```bash
# Step 1：找到 CPU 最高的 Java 进程
top -c | grep java

# Step 2：找到该进程中 CPU 最高的线程
top -H -p <PID>
# 记录线程 ID（十进制），如 12345

# Step 3：转换为十六进制
printf '%x\n' 12345  # → 3039

# Step 4：jstack 找对应线程
jstack <PID> | grep "3039" -A 20

# 常见原因：
# - 无限循环（循环条件 bug）
# - JVM Full GC 导致 GC 线程占用（jstat 确认）
# - 正则表达式回溯（复杂正则 + 特殊输入）
# - 序列化/反序列化密集操作
```

```bash
# Arthas 更方便（无需手动换算）
java -jar arthas-boot.jar
thread -n 5        # 找 CPU 最高的 5 个线程及其堆栈
thread -b          # 找死锁线程
```

### 4.3 OOM 排查

```bash
# OOM 类型及排查方向：

# 1. Java heap space → 堆内存不够
java -XX:+HeapDumpOnOutOfMemoryError ...
# 分析 heap dump（MAT），找大对象/泄漏

# 2. Metaspace → 元空间不够
# 原因：大量动态生成类（Dubbo、CGLib 代理）
-XX:MaxMetaspaceSize=512m  # 设置上限
# 用 jmap -clstats <PID> 查看类加载统计

# 3. Direct buffer memory → 直接内存不够
# 原因：Netty 直接内存未释放、NIO 操作大量数据
-XX:MaxDirectMemorySize=1g  # 设置上限

# 4. GC overhead limit exceeded
# 原因：GC 时间占比 > 98% 且每次回收 < 2% 内存
# 说明内存真的不够用，加堆或排查泄漏

# 5. unable to create new native thread
# 原因：线程数超限（ulimit -u 或系统线程数上限）
# 解决：减少线程数，或增大 ulimit，或 JDK 21 虚拟线程
```

### 4.4 内存泄漏排查（jmap + MAT）

```bash
# 1. 触发 Full GC 后 dump（排除只是缓存未 GC 的干扰）
jmap -histo:live <PID> | head -50

# 2. dump 堆快照
jmap -dump:live,format=b,file=/tmp/heap.hprof <PID>
# 注意：-dump:live 会触发 Full GC，生产环境谨慎

# 3. MAT 分析步骤：
#    File → Open Heap Dump
#    → Reports → Leak Suspects  ← 自动识别泄漏嫌疑
#    → Dominator Tree           ← 找最占内存的对象树
#    → OQL Console              ← SQL 方式查询对象

# 4. 常用 OQL
SELECT * FROM java.util.HashMap WHERE size > 10000
SELECT * FROM java.lang.Thread
```

---

## 五、Arthas 实战

```bash
# 下载并启动
curl -O https://arthas.aliyun.com/arthas-boot.jar
java -jar arthas-boot.jar

# 常用命令
dashboard        # 实时面板（线程、内存、GC）
thread -n 3      # CPU 最高的3个线程
thread -b        # 检测死锁
jad com.example.UserService  # 反编译类（确认线上代码版本）
watch com.example.UserService queryUser "{params,returnObj}" -x 2  # 监控方法入参出参
trace com.example.UserService queryUser  # 方法调用链路追踪（找慢方法）
monitor -c 5 com.example.UserService queryUser  # 5秒统计方法调用次数/耗时
ognl "@System@currentTimeMillis()"  # 执行 OGNL 表达式
heapdump /tmp/heap.hprof             # dump 堆
```

---

## 六、调优案例

### 案例1：新生代太小导致频繁 Young GC

```
问题：jstat 显示每秒触发 5 次以上 Young GC
分析：Eden 区太小，对象分配速率高
解决：增大新生代（-Xmn2g 或 -XX:G1NewSizePercent=30）
```

### 案例2：大对象导致频繁 Full GC

```
问题：G1 日志中大量 Humongous Allocation，频繁 Full GC
分析：对象大小 > Region 大小的 50%，直接进老年代
解决：
  a. 增大 Region 大小（-XX:G1HeapRegionSize=16m）
  b. 优化代码，减少大对象（如：分批查询代替全量查询）
```

### 案例3：元空间 OOM

```
问题：服务运行一段时间后 Metaspace OOM
分析：jmap -clstats 显示类数量持续增长
原因：JSP 热部署未卸载旧类；或 CGLib 大量生成代理类未回收
解决：
  a. 设置 -XX:MaxMetaspaceSize 限制上限（触发 GC 回收无用类）
  b. 排查类加载器泄漏（MAT 查 ClassLoader 引用树）
```

---

## 七、面试要点速查

| 问题 | 要点 |
|------|------|
| CPU 飙高如何排查 | top → 找线程 ID → printf '%x' → jstack 找对应线程的堆栈 |
| OOM 如何排查 | -HeapDumpOnOutOfMemoryError 自动 dump → MAT 分析 Leak Suspects |
| 如何定位内存泄漏 | jmap -histo:live 对比多次结果；MAT Dominator Tree |
| jstat 的作用 | 实时监控 GC 次数/耗时、各内存区域使用率（不需要停服）|
| 为什么 Xms 要等于 Xmx | 避免堆动态扩容引起的 GC 停顿 |
| Full GC 频繁的原因 | 内存泄漏/老年代太小/大对象/元空间不足/System.gc() 调用 |
| Arthas 能做什么 | 在线诊断（方法追踪、入参出参监控、反编译、死锁检测），无需重启 |


---

**相关面试题** → [[../../10_Developlanguage/001_java/04_JavaJVMSubject/06、JVM 调优与参数|06、JVM 调优与参数]] | [[../../10_Developlanguage/001_java/04_JavaJVMSubject/10、面试实战与案例分析|10、面试实战与案例分析]]
