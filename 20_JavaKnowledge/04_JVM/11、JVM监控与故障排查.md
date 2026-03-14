# JVM 监控与故障排查

> **核心关键词**：jps、jstat、jmap、jstack、jcmd、JFR、JMC、Arthas、Prometheus、故障排查流程

---

## 一、JDK 内置命令行工具

### 1.1 jps — 查看 Java 进程

```bash
jps           # 列出所有 Java 进程的 PID 和主类名
jps -l        # 显示完整主类名/JAR 路径
jps -v        # 显示 JVM 启动参数
jps -m        # 显示主类的 main 方法参数
```

### 1.2 jstat — 实时监控 GC 统计

```bash
# 格式：jstat -<option> <PID> [interval] [count]
jstat -gc <PID> 1000 10          # 每秒打印一次 GC 统计，共10次

# 输出各列含义：
# S0C S1C：Survivor 0/1 区容量（KB）
# S0U S1U：Survivor 0/1 区已用（KB）
# EC EU：Eden 容量/已用
# OC OU：Old 区容量/已用
# MC MU：元空间容量/已用
# YGC YGCT：Young GC 次数/总耗时
# FGC FGCT：Full GC 次数/总耗时

jstat -gcutil <PID> 1000         # 百分比形式（更直观）
jstat -gccapacity <PID>          # 各代容量
jstat -gcnew <PID>               # 新生代统计
jstat -gcold <PID>               # 老年代统计
jstat -class <PID>               # 类加载统计
jstat -compiler <PID>            # JIT 编译统计

# 实战：判断 GC 是否有问题
# - YGC 频率 > 1次/秒：Young GC 太频繁，考虑增大 Eden
# - FGC 持续增加：Full GC 频繁，排查内存泄漏或堆太小
# - OU 持续增长不回落：老年代内存泄漏
```

### 1.3 jmap — 堆内存分析

```bash
# 查看堆概要信息
jmap -heap <PID>
# JDK 9+ 用 jhsdb
jhsdb jmap --heap --pid <PID>

# 查看对象统计（按实例数/占用内存排序）
jmap -histo <PID> | head -30     # 所有对象
jmap -histo:live <PID> | head -30 # 存活对象（先触发 Full GC）

# dump 堆快照（生产环境慎用，会暂停 JVM！）
jmap -dump:format=b,file=/tmp/heap.hprof <PID>
jmap -dump:live,format=b,file=/tmp/heap_live.hprof <PID>  # 只 dump 存活对象

# JDK 9+ 推荐用 jcmd
jcmd <PID> GC.heap_dump /tmp/heap.hprof

# 类加载器统计（排查元空间泄漏）
jmap -clstats <PID>
```

### 1.4 jstack — 线程转储

```bash
# 打印所有线程的堆栈信息
jstack <PID>
jstack -l <PID>    # 包含锁信息（建议加 -l）

# 输出到文件
jstack -l <PID> > /tmp/thread_dump.txt

# 自动检测死锁（在输出末尾）
jstack <PID> | grep -A 30 "Found.*deadlock"

# 统计线程状态分布
jstack <PID> | grep "java.lang.Thread.State" | sort | uniq -c | sort -rn
# BLOCKED 状态多 → 锁竞争
# WAITING/TIMED_WAITING 多 → 正常（线程池等待任务）
# RUNNABLE 多 → 正常执行或 CPU 密集

# JDK 9+ 用 jcmd
jcmd <PID> Thread.print -l
```

### 1.5 jcmd — 多功能诊断工具（推荐）

```bash
# 列出所有可用命令
jcmd <PID> help

# 常用命令
jcmd <PID> VM.version          # JVM 版本
jcmd <PID> VM.flags            # JVM 参数
jcmd <PID> VM.system_properties # 系统属性
jcmd <PID> VM.native_memory    # 本地内存统计（需 -XX:NativeMemoryTracking=summary）
jcmd <PID> GC.run              # 触发 GC
jcmd <PID> GC.heap_info        # 堆信息
jcmd <PID> GC.heap_dump /tmp/heap.hprof  # heap dump
jcmd <PID> Thread.print -l     # 线程 dump
jcmd <PID> JFR.start duration=60s filename=/tmp/rec.jfr  # 启动 JFR 录制
```

### 1.6 jinfo — 查看/修改 JVM 参数

```bash
jinfo -flags <PID>           # 查看 JVM 参数
jinfo -flag MaxHeapSize <PID> # 查看特定参数
jinfo -flag +PrintGCDetails <PID>  # 动态开启 GC 日志（部分参数支持）
```

---

## 二、JFR（Java Flight Recorder）

```java
// JFR：JDK 11+ 免费使用的低开销生产级诊断工具
// 持续记录 JVM 内部事件（GC、线程、IO、异常、方法调用等）
// 开销 < 1%（与 async-profiler 类似）

// 启动时开启 JFR
java -XX:StartFlightRecording=duration=60s,filename=/tmp/app.jfr MyApp

// 运行时开启
jcmd <PID> JFR.start name=MyRecording duration=120s filename=/tmp/app.jfr
jcmd <PID> JFR.check         # 查看录制状态
jcmd <PID> JFR.stop name=MyRecording  # 停止录制

// 分析工具：JMC（Java Mission Control）
// 打开 .jfr 文件，可分析：
//   - 方法热点（CPU 采样）
//   - GC 行为（各阶段耗时）
//   - 锁竞争（等待时间）
//   - 内存分配（对象分配热点）
//   - IO 等待
```

---

## 三、Arthas 深度使用

```bash
# 启动
java -jar arthas-boot.jar  # 选择进程后进入交互界面

# ① 实时诊断面板
dashboard  # CPU/内存/GC/线程一览

# ② 线程分析
thread -n 5        # Top 5 CPU 线程
thread 12345       # 查看指定线程详情
thread -b          # 死锁检测
thread -s BLOCKED  # 过滤 BLOCKED 线程

# ③ 方法监控
# watch：监控方法的入参/出参/异常/耗时
watch com.example.UserService queryUser '{params, returnObj, throwExp}' -x 3
# -x：结果展开深度
# -e：只在异常时触发
watch com.example.UserService queryUser '{params}' -e -x 2

# ④ 性能追踪
trace com.example.UserService queryUser   # 方法调用链耗时
trace -E com.example.Service 'method1|method2'  # 正则匹配多个方法
stack com.example.Cache get               # 查看调用来源（调用栈）

# ⑤ 统计监控
monitor -c 5 com.example.UserService queryUser  # 5秒统计：调用次数/成功率/均值/P95

# ⑥ 类操作
jad com.example.UserService             # 反编译（查看线上实际运行的代码）
sc -d com.example.UserService           # 查看类的详细信息
sm com.example.UserService queryUser    # 查看方法签名

# ⑦ 热更新（谨慎！）
mc /tmp/UserService.java                # 内存编译
redefine /tmp/com/example/UserService.class  # 热加载（不影响类结构的修改）

# ⑧ 内存与 GC
memory          # 各内存区域使用量
heapdump /tmp/arthas_heap.hprof
ognl '@java.lang.Runtime@getRuntime().totalMemory()'  # OGNL 表达式

# ⑨ 退出
stop   # 完全退出 Arthas（恢复原 JVM 状态）
```

---

## 四、可视化监控方案

### 4.1 Prometheus + Grafana + JMX Exporter

```yaml
# JMX Exporter 配置（jmx_config.yaml）
rules:
  - pattern: "java.lang<type=Memory><HeapMemoryUsage>used"
    name: jvm_memory_used_bytes
  - pattern: "java.lang<type=GarbageCollector,name=G1 Young Generation><CollectionCount>"
    name: jvm_gc_collection_count_total

# 启动时加 agent
java -javaagent:/opt/jmx_exporter.jar=8090:/opt/jmx_config.yaml -jar app.jar

# Grafana 仪表盘：推荐使用 JVM Overview 仪表盘（Dashboard ID: 4701）
```

### 4.2 关键监控指标

```
堆内存：
  heap_used / heap_max → 使用率（>80% 报警）
  old_gen_used 趋势 → 持续增长说明泄漏

GC 指标：
  ygc_count / ygc_time → Young GC 频率/耗时
  fgc_count / fgc_time → Full GC 次数（>0 触发报警）
  gc_pause_p99 → GC 停顿时间 P99

线程：
  thread_count_total → 线程总数（异常增长报警）
  thread_blocked → BLOCKED 线程数

类加载：
  classes_loaded → 已加载类数量（持续增长排查 MetaSpace 泄漏）
```

---

## 五、故障排查标准流程

```
问题：服务响应缓慢 / 无响应 / OOM

Step 1：快速定位问题类型
  jstat -gcutil <PID> 1000
  → FGC 持续增加？→ Full GC 问题
  → OU 接近 OC？ → 老年代快满
  → YGC 极频繁？ → 新生代太小/分配速率高

Step 2：CPU 飙高时
  top → thread -n 5 (Arthas) → 查看线程堆栈
  jstack <PID> → 分析是否 GC 线程(VM Thread)占 CPU

Step 3：内存问题
  jmap -histo:live <PID> | head -30 → 找大对象
  jmap -dump:live,format=b,file=heap.hprof <PID>
  MAT → Leak Suspects / Dominator Tree

Step 4：线程 / 死锁问题
  jstack -l <PID>
  Arthas: thread -b

Step 5：代码问题
  Arthas: jad / watch / trace
  确认线上运行的是预期版本的代码
  追踪方法调用链找到慢点

Step 6：恢复 & 根因分析
  记录现场数据（日志、GC日志、thread dump、heap dump）
  重启服务恢复
  分析根因，修复代码或调整参数
```

---

## 六、面试要点速查

| 问题 | 要点 |
|------|------|
| jstat 最常用的选项 | `-gcutil`（百分比），`-gc`（字节）；看 YGC/FGC 次数和时间 |
| 如何在不重启的情况下诊断 | jstack（线程）、jmap（堆）、Arthas（方法追踪、反编译、热更新）|
| JFR 的优势 | 生产级低开销（<1%），持续录制，事后分析，无需复现问题 |
| CPU 飙高排查思路 | top → jstack → 找 nid 匹配的线程 → 分析堆栈；或用 Arthas thread -n |
| heap dump 的影响 | jmap -dump 需要 STW，生产环境谨慎；推荐用 -HeapDumpOnOutOfMemoryError 自动触发 |
| 如何判断是否内存泄漏 | jstat 观察 OU 趋势；Full GC 后 OU 不降；jmap histo 对比多次结果 |


---

**相关面试题** → [[../../10_Developlanguage/001_java/04_JavaJVMSubject/07、性能监控与故障排查|07、性能监控与故障排查]]
