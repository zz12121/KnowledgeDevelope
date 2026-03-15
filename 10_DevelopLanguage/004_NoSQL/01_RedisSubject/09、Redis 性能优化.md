###### 1. 如何优化 Redis 的性能?
Redis 性能优化是系统工程，需要从架构、配置、代码、数据模型多个维度入手。

**架构层面**：使用连接池，避免频繁创建销毁连接；读压力大时上主从复制 + 读写分离；写压力或数据量超出单机时上集群；在 Redis 前加本地缓存（Caffeine），热点数据不走网络。

**配置层面**：调整数据结构编码阈值（如 `hash-max-listpack-entries`），让小数据使用更紧凑的 listpack 编码；启用 `activedefrag yes` 做主动内存碎片整理；根据数据重要性合理配置持久化（纯缓存可以只用 RDB，重要数据用混合持久化）。

**命令层面**：使用 Pipeline 批量发送命令，减少 RTT；避免大 Key 操作（阻塞主线程）；禁用 `KEYS *` 等全量扫描命令，改用 `SCAN`；删除大 Key 用 `UNLINK` 而不是 `DEL`。

**数据模型层面**：选择合适的数据结构，利用 Redis 自动编码优化（比如小 ZSet 会用 listpack 而不是跳跃表，内存节省很多）；对大 Key 拆分，降低单个 key 的数据量。

**根本原理**：Redis 核心是单线程事件循环，任何慢操作都会阻塞整个服务，优化的核心就是**不让主线程做慢的事**。

> 📖 [[../../../23_RedisKnowledge/09_性能优化/01、性能优化与网络模型|知识库：性能优化与网络模型]]

###### 2. Redis 的慢查询日志是什么?
慢查询日志记录执行时间超过阈值的命令，是性能排查最直接的工具。

**关键点**：记录的是命令在 Redis **服务端实际执行时间**，不含网络传输和排队等待时间。如果 `redis-cli --latency` 发现延迟高但慢日志里没啥，说明瓶颈在网络，不在命令执行。

**配置与使用**：

```bash
# 设置阈值（微秒）和最大记录数
CONFIG SET slowlog-log-slower-than 5000  # 超过 5ms 记录
CONFIG SET slowlog-max-len 1000
CONFIG REWRITE  # 持久化到配置文件

SLOWLOG GET 10   # 查看最近 10 条慢查询
SLOWLOG LEN      # 慢查询总数
SLOWLOG RESET    # 清空
```

**生产建议**：高并发场景将阈值设为 1ms（1000 微秒），`slowlog-max-len` 设 1000 以上，通过监控系统定期采集 `SLOWLOG GET`，发现突增时及时排查。

> 📖 [[../../../23_RedisKnowledge/09_性能优化/01、性能优化与网络模型#8-慢查询日志|知识库：慢查询日志配置]]

###### 3. 如何分析 Redis 的性能瓶颈?
性能分析要有系统性，分几个层次来排查。

**第一步：监控基础指标**

```bash
redis-cli info all  # 全量信息
redis-cli info stats | grep -E 'ops_per_sec|hits|misses'  # QPS 和命中率
redis-cli info memory | grep used_memory_human  # 内存使用
redis-cli info clients | grep connected_clients  # 连接数
```

命中率 `keyspace_hits / (keyspace_hits + keyspace_misses)` 低说明缓存设计有问题；`aof_delayed_fsync` 高说明 AOF 持久化在拖累性能。

**第二步：分析慢查询日志**

`SLOWLOG GET 20` 直接看哪些命令慢，这是定位问题最直接的方式。

**第三步：延迟诊断**

```bash
redis-cli --latency          # 实时延迟
redis-cli --latency-history  # 历史延迟（15 秒一次）
redis-cli --stat             # 实时统计（内存/QPS/命中率）
```

**第四步：压测找瓶颈**

```bash
redis-benchmark -h 127.0.0.1 -p 6379 -c 50 -n 100000  # 50 并发 10 万次请求
```

**常见瓶颈根源**：`KEYS *` 等全量命令阻塞；大 Key 的操作；持久化 fork 阻塞（`latest_fork_usec` 过高）；内存接近上限触发内存淘汰；网络带宽打满。

> 📖 [[../../../23_RedisKnowledge/09_性能优化/01、性能优化与网络模型|知识库：性能瓶颈分析]]

###### 4. Pipeline 是什么?有什么优势?
Pipeline 是一种**客户端技术**，把多个命令打包成一批，一次性发给 Redis，服务器按顺序执行完后把所有结果一次性返回。

**本质是减少网络往返（RTT）**：

- 普通模式：发命令1 → 收响应1 → 发命令2 → 收响应2 → ...（N 次 RTT）
- Pipeline 模式：批量发（命令1 + 命令2 + 命令3）→ 批量收（响应1 + 响应2 + 响应3）（1 次 RTT）

在网络延迟 1ms 的场景下，100 条命令普通模式需要至少 100ms，Pipeline 只需 1ms + 执行时间。

**Java 示例（Jedis）**：

```java
Pipeline p = jedis.pipelined();
p.set("key1", "val1");
p.set("key2", "val2");
Response<String> r1 = p.get("key1");  // 返回的是 Future，此时还没发送
p.sync();  // 这里才真正批量发送并等待所有结果
System.out.println(r1.get());  // 现在可以取值
```

**注意**：Pipeline **没有原子性保证**，命令间不能有依赖关系。需要原子性用 `MULTI/EXEC` 事务；需要有依赖逻辑（上一条的结果决定下一条）用 Lua 脚本。

> 📖 [[../../../23_RedisKnowledge/09_性能优化/01、性能优化与网络模型#4-pipeline|知识库：Pipeline 原理与使用]]

###### 5. Redis 的批量操作有哪些?
Redis 原生支持一些批量命令，可以一次操作多个 key 或多个字段：

- `MSET key1 val1 key2 val2 ...` / `MGET key1 key2 ...`：批量操作 String
- `HMGET key field1 field2 ...` / `HMSET key field1 val1 ...`：批量操作 Hash 字段
- `SADD key member1 member2 ...`：一次添加多个 Set 成员
- `ZADD key score1 m1 score2 m2 ...`：一次添加多个 ZSet 成员
- `LPUSH / RPUSH key val1 val2 ...`：一次向 List 插入多个元素

**原生批量命令 vs Pipeline 如何选择**？原生批量命令是原子的（`MSET` 要么全成功要么全失败），适合操作同一数据结构的多个值；Pipeline 更灵活，适合批量执行不同类型的命令，但没有原子性保证。能用原生批量命令解决的优先用原生批量命令。

> 📖 [[../../../23_RedisKnowledge/09_性能优化/01、性能优化与网络模型#4-pipeline|知识库：Pipeline 与批量操作]]

###### 6. 大 key 问题是什么?如何处理?
大 Key 通常指 String value > 10KB，或者 Hash/List/Set/ZSet 的元素数量 > 5000 的 key。

**危害**：操作大 Key 会长时间占用主线程（`DEL` 一个 100MB 的 key 可能需要几十毫秒），阻塞后续所有请求；传输大 Key 占用大量网络带宽；集群中大 Key 所在节点内存远超其他节点，导致数据不均衡；fork 子进程持久化时写时复制压力大。

**发现方法**：

```bash
redis-cli --bigkeys         # 扫描找出各类型最大的 key（会扫全库，低峰期执行）
MEMORY USAGE mykey          # 查看特定 key 的内存占用
OBJECT ENCODING mykey       # 查看编码，判断是否能优化
```

**处理方案**：

- **拆分**：大 Hash 按字段哈希取模拆成多个小 Hash（`user:info:1000` → `user:info:1000:{0~9}`）；大 List/Set 按元素数量拆分
- **压缩**：JSON 或文本类 value 在客户端压缩后存储（gzip/snappy），取出后解压
- **渐进式删除**：不要一次性 `DEL`，先用 `HSCAN`/`SSCAN` 分批删子元素，最后再 `DEL` 空壳
- **异步删除**：改用 `UNLINK` 命令，主线程只做逻辑删除，实际内存回收由后台线程异步完成

> 📖 [[../../../23_RedisKnowledge/09_性能优化/01、性能优化与网络模型#5-大-key-问题|知识库：大 Key 问题处理]]

###### 7. 热 key 问题是什么?如何处理?
热 Key 是访问频率远超普通 Key 的特定 key（比如某个明星的微博、秒杀商品详情），可能导致所在 Redis 节点成为瓶颈。

**危害**：单节点 CPU/网络打满，影响该节点上其他 key 的响应；热 Key 过期时可能引发缓存击穿；集群中热 Key 所在节点负载远超其他节点。

**发现方法**：

```bash
redis-cli --hotkeys         # Redis 4.0+ 发现热 key（需 maxmemory-policy 非 none）
```

也可以在客户端中间件层统计 key 访问频率，更灵活。

**处理方案**：

**本地缓存**：在应用层用 Caffeine 缓存热 Key，命中本地缓存就不走 Redis，彻底消除网络开销。注意设置合理的本地缓存 TTL，避免数据过期不同步。

**逻辑分片**：把一个热 Key 复制成 N 份，如 `hotkey:0` 到 `hotkey:9`，访问时随机选一份，把压力分摊到 N 个 slot（在集群中可能分布到多个节点）。更新时 N 份都要更新。

**永不过期 + 异步刷新**：热 Key 不设 TTL，后台定时任务更新内容，彻底避免击穿。

> 📖 [[../../../23_RedisKnowledge/09_性能优化/01、性能优化与网络模型#6-热-key-问题|知识库：热 Key 处理策略]]

###### 8. Redis 的 Lazy Free 是什么?
Lazy Free 是 Redis 4.0 引入的特性，解决了**删除大 Key 阻塞主线程**的问题。

**核心思路**：把"删除 key 的内存回收"从主线程同步完成，改为**主线程只做逻辑标记，后台 BIO 线程异步回收内存**。这样主线程的 DEL/过期清理等操作几乎不耗时，不会阻塞服务。

**触发方式**：
- `UNLINK key`：异步版的 `DEL`，主线程立即返回，内存由后台线程回收
- `FLUSHDB ASYNC` / `FLUSHALL ASYNC`：异步清库
- 配置开关，让更多场景使用 Lazy Free：

```bash
lazyfree-lazy-eviction yes    # 内存满淘汰时异步删除
lazyfree-lazy-expire yes      # 过期 key 异步删除
lazyfree-lazy-server-del yes  # 某些写命令覆盖旧值时异步删除旧值
```

**生产建议**：所有 Lazy Free 配置建议都开启，配合 `UNLINK` 替代 `DEL` 来删除大 Key，几乎不增加主线程延迟。

> 📖 [[../../../23_RedisKnowledge/09_性能优化/01、性能优化与网络模型#7-lazy-free|知识库：Lazy Free 机制]]

###### 9. 如何避免 Redis 的阻塞操作?
Redis 是单线程的（命令执行），任何慢操作都会造成其他请求排队，所以避免阻塞是性能优化的核心。

**替换危险命令**：

| 危险命令 | 替代方案 |
|---------|---------|
| `KEYS *` | `SCAN count 100`（渐进式扫描） |
| `SMEMBERS` 全量 | `SSCAN` 分批扫描 |
| `HGETALL` 大 Hash | `HSCAN` 分批扫描 |
| `DEL` 大 Key | `UNLINK` 异步删除 |
| `FLUSHDB/FLUSHALL` | `FLUSHDB ASYNC` |

**控制命令粒度**：不要一次性 `LPUSH` 几万个元素，分批写入；Sorted Set 的 `ZUNIONSTORE`/`ZINTERSTORE` 大集合运算很慢，考虑离线计算。

**谨慎使用 Lua 脚本**：Lua 脚本执行期间主线程被占用，脚本里不能有长循环，不能有等待（Lua 里调不了 `WAIT` 这类命令），执行时间控制在毫秒级以内。

**持久化影响**：AOF `always` 策略每次写都 fsync，不建议生产用；RDB/AOF 重写的 fork 操作会短暂阻塞，控制实例内存大小（建议 < 20GB），并在低峰期触发。

> 📖 [[../../../23_RedisKnowledge/09_性能优化/01、性能优化与网络模型#3-阻塞操作替代方案|知识库：阻塞操作替代方案]]

###### 10. Redis 的网络模型是什么?
Redis 的网络模型是高性能的基石，核心是**单线程 Reactor 模型**。

**核心组件**：
1. **I/O 多路复用器**：Redis 封装了 Linux 的 `epoll`（macOS 用 `kqueue`，BSD 用 `select`），一个线程同时监听成千上万个 socket 的读写事件，不需要为每个连接创建线程
2. **事件分发器**：I/O 多路复用器发现事件后，通知对应的事件处理器
3. **事件处理器**：包括连接应答处理器（新连接）、命令请求处理器（读命令）、命令回复处理器（发结果）

**工作流程**（源码 `ae.c`）：

```
aeMain() 循环
  └── aeProcessEvents()
       └── epoll_wait()  ← 等待有 socket 事件
           当有事件时：
           ├── 新连接 → acceptTcpHandler() → 创建客户端
           ├── 可读  → readQueryFromClient() → 解析命令
           │             → processCommand()  → 执行命令
           └── 可写  → sendReplyToClient()   → 发送响应
```

所有命令在单线程里顺序执行，天然无锁，不需要加任何 mutex，是 Redis 实现简单高效的根本原因。

**Redis 6.0 的演进**：引入了多线程 I/O，让 `readQueryFromClient`（读取请求）和 `sendReplyToClient`（发送响应）可以并行执行，命令的实际处理（`processCommand`）仍然是单线程串行。在返回数据量很大的场景下，多线程 I/O 能明显提升吞吐量。

> 📖 [[../../../23_RedisKnowledge/09_性能优化/01、性能优化与网络模型#1-单线程-reactor-模型|知识库：Redis 网络模型详解]]
