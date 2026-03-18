---
tags:
  - Java/JVM
  - Java/GC
  - Java/ZGC
aliases:
  - ZGC
  - Shenandoah
  - 低延迟GC
date: 2026-03-18
---

# ZGC 与 Shenandoah

> **核心关键词**：低延迟 GC、着色指针、读屏障、并发整理、停顿 < 10ms、可伸缩、染色指针

---

## 一、为什么需要更低延迟的 GC

```
GC 停顿时间演进：
  Serial GC       → 几百ms ~ 几秒（STW 单线程）
  Parallel GC     → 几十ms ~ 几百ms（STW 多线程）
  CMS             → 几十ms（并发标记，但碎片化）
  G1              → 100ms ~ 200ms（可配置，区域化）
  ZGC / Shenandoah → < 10ms（几乎全程并发，与堆大小无关！）

应用场景：
  延迟敏感型服务（交易系统、游戏、实时推荐）
  超大堆（TB 级别）
  要求 99.9th percentile 延迟 < 10ms 的服务
```

---

## 二、ZGC（Z Garbage Collector）

### 2.1 核心设计目标

```
1. 停顿时间 < 1ms（JDK 16+ 进一步优化）/ < 10ms（通常）
2. 停顿时间不随堆大小增长（可扩展到 TB 级堆）
3. 吞吐量不低于 G1 的 15%（实际生产中通常 < 5% 差距）
4. JDK 11 实验性引入，JDK 15 正式 Production Ready
5. JDK 21 支持分代 ZGC（Generational ZGC），吞吐量大幅提升
```

### 2.2 核心技术：着色指针（Colored Pointers）

```
ZGC 的革命性创新：将 GC 元数据编码到 64位指针的高位

┌──────────────────────────────────────────────────────┐
│  6位：未用 │ 4位：颜色标记 │         42位：对象地址       │
└──────────────────────────────────────────────────────┘

4位颜色标记含义：
  Marked0 (M0)  : 本轮 GC 标记颜色（交替使用）
  Marked1 (M1)  : 上轮 GC 标记颜色
  Remapped (R)  : 已重映射（指向对象的最新地址）
  Finalizable   : 只有 finalizer 可达

着色指针的优势：
  1. 无需额外的侧表（Side Table）记录对象状态
  2. 标记和重映射通过修改指针颜色完成
  3. 读屏障可以通过颜色快速判断指针是否需要修复
```

### 2.3 核心技术：读屏障（Load Barrier）

```java
// 传统 GC（写屏障）：对象引用被修改时触发
// ZGC（读屏障）：从堆中加载对象引用时触发

// 伪代码：ZGC 读屏障
Object loadBarrier(Object* ref) {
    Object obj = *ref;
    if (obj.color != GOOD_COLOR) {  // 如果指针颜色"不好"
        obj = fixPointer(ref, obj); // 修复指针（重映射到新地址）
    }
    return obj;
}

// 读屏障的作用：
// 1. 标记阶段：标记对象（设置 Marked 颜色）
// 2. 转移阶段：将指针重映射到对象的新地址（完成并发移动）

// 代价：每次对象读取都有微小开销（约 3-5% 吞吐量损失）
// 比 GC 停顿 STW 代价小得多
```

### 2.4 ZGC 的 GC 周期

```
ZGC 几乎所有阶段都并发进行（与用户线程同时）：

阶段               是否 STW   说明
─────────────────────────────────────────────
Pause Mark Start   ✅ STW     扫描 GC Roots（< 1ms）
Concurrent Mark    ✗ 并发     标记所有可达对象（读屏障协助）
Pause Mark End     ✅ STW     处理标记结束时的变更（< 1ms）
Concurrent Prepare ✗ 并发     准备重定位集合（选择要转移的 Region）
Pause Relocate     ✅ STW     扫描 GC Roots（< 1ms）
Concurrent Relocate ✗ 并发   并发转移对象（旧地址留转发表）
Concurrent Remap   ✗ 并发     修复所有旧指针（可推迟到下轮）
─────────────────────────────────────────────
STW 次数：3次，每次 < 1ms，与堆大小无关！
```

### 2.5 ZGC 关键参数

```bash
# 启用 ZGC
-XX:+UseZGC

# 分代 ZGC（JDK 21+，推荐）
-XX:+UseZGC -XX:+ZGenerational

# 停顿时间目标（ZGC 参考但不严格保证）
-XX:SoftMaxHeapSize=4g         # 软堆上限（触发 GC 的目标）

# 并发 GC 线程数（默认根据 CPU 数量计算）
-XX:ConcGCThreads=4

# 触发阈值（堆分配速率 vs GC 速率）
-XX:ZAllocationSpikeTolerance=2.0  # 容忍分配峰值倍数

# 日志
-Xlog:gc*:file=gc.log
```

---

## 三、Shenandoah GC（Red Hat）

### 3.1 核心设计

```
Shenandoah：Red Hat 开发，JDK 12 实验，JDK 15 正式 (非 OpenJDK 主线，OpenJDK 14+ 支持)
目标：与 ZGC 类似，停顿时间极短，并发整理

核心技术与 ZGC 的区别：
  ZGC：着色指针（指针中存颜色）
  Shenandoah：转发指针（Brooks Pointer，对象头中存转发地址）
```

### 3.2 Brooks 指针（Forwarding Pointer）

```
每个对象额外加一个指针字段（Brooks Pointer）：
  正常状态：指针指向对象本身
  并发转移时：指针指向新地址

┌─────────────────────┐
│  Brooks Pointer      │  → 初始指向自己，转移后指向新地址
├─────────────────────┤
│  对象头              │
├─────────────────────┤
│  实例数据            │
└─────────────────────┘

读屏障：访问对象时先解引用 Brooks Pointer，确保读到最新地址
写屏障：修改对象时需 CAS 更新 Brooks Pointer（并发转移的核心）

代价：每个对象额外一个指针（8字节），内存开销大于 ZGC
```

### 3.3 Shenandoah GC 周期

```
阶段                    是否 STW
──────────────────────────────────
Initial Mark            ✅ STW（很短）
Concurrent Mark         ✗ 并发
Final Mark              ✅ STW（很短）
Concurrent Cleanup      ✗ 并发
Concurrent Evacuation   ✗ 并发（并发移动对象！）
Init Update Refs        ✅ STW（很短）
Concurrent Update Refs  ✗ 并发（并发更新引用）
Final Update Refs       ✅ STW（很短）
Concurrent Cleanup      ✗ 并发
```

```bash
# 启用 Shenandoah
-XX:+UseShenandoahGC

# 触发策略
-XX:ShenandoahGCHeuristics=adaptive  # 自适应（默认）
# 其他选项：static, compact, aggressive
```

---

## 四、三者对比：ZGC vs Shenandoah vs G1

| 对比维度 | G1 | ZGC | Shenandoah |
|---------|-----|-----|-----------|
| 停顿时间 | 100-200ms | < 1ms | < 5ms |
| 停顿时间与堆大小关系 | 相关 | **无关** | **无关** |
| 并发整理 | 否 | ✅ | ✅ |
| 内存开销 | 低（RSet） | 低（指针着色） | 较高（Brooks指针）|
| 吞吐量 | 高 | 略低（< 5%） | 略低 |
| 适用堆大小 | 4GB-32GB | **8GB-TB** | 2GB-TB |
| 维护方 | Oracle | Oracle | Red Hat |
| JDK 版本（生产可用） | JDK 7+ | **JDK 15+** | JDK 15+ |
| 默认 GC | JDK 9-20 | JDK 21+ 推荐 | 非 Oracle JDK |

---

## 五、选型建议

```
普通 Web 服务（< 8GB 堆，延迟要求一般）→ G1（默认）

延迟敏感（响应时间 P99 < 100ms），堆 < 32GB → ZGC

超大堆（> 32GB），TB 级数据缓存服务 → ZGC

使用 Red Hat 系（OpenShift/RHEL），对 Shenandoah 有运维经验 → Shenandoah

JDK 21+ 新项目 → 考虑分代 ZGC（-XX:+UseZGC -XX:+ZGenerational）
```

---

## 六、面试要点速查

| 问题 | 要点 |
|------|------|
| ZGC 停顿为什么这么短 | 几乎所有阶段并发（与用户线程同时），STW 只有3次短暂的 Root 扫描 |
| 着色指针的作用 | 将 GC 状态（标记/转移/重映射）编码进指针高位，读屏障据此修复指针 |
| ZGC 和 G1 的根本区别 | G1 并发标记但整理是 STW；ZGC 连整理（对象转移）都是并发的 |
| ZGC 停顿时间随堆大小增长吗 | 不增长，STW 只扫描 GC Roots，与堆大小无关 |
| Shenandoah 的核心技术 | Brooks 转发指针（对象头多一个指向自己/新地址的指针）|
| 分代 ZGC（JDK 21）的意义 | 引入分代，减少全堆扫描，大幅提升吞吐量（接近 G1）|


---

**相关面试题** → [[../../10_Developlanguage/001_Java/04_JavaJVMSubject/05、新一代垃圾收集器|05、新一代垃圾收集器]]
