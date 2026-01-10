###### 1. 如何优化 Redis 的性能?
Redis性能优化是一个系统工程，需从架构、配置、代码、数据模型等多维度入手。
**1. 架构与部署优化**
- **使用连接池**：避免频繁创建销毁连接。使用Jedis或Lettuce等客户端，并合理配置最大/最小空闲连接数。
- **部署模式选择**：读写压力大或数据量大时，采用**主从复制+哨兵**或**Redis Cluster集群**，实现读写分离与数据分片。
- **多级缓存架构**：在Redis前加入本地缓存（如Caffeine），减少网络IO。将高频读取但极少变更的数据（如字典数据）存入本地缓存。
**2. 配置优化**
- **内存优化**：
    - 根据业务特点调整数据结构的编码阈值（如`hash-max-listpack-entries`, `zset-max-listpack-entries`），让小数据使用更紧凑的`listpack`（Redis 7+）或`ziplist`编码，降低内存占用。
    - 启用`activedefrag yes`进行主动内存碎片整理，当碎片率（`mem_fragmentation_ratio`）超过1.5时尤其有效。
- **持久化优化**：根据数据重要性选择RDB、AOF或混合模式。对性能要求极高且可容忍数据丢失的场景，可仅在从节点开启AOF。
**3. 客户端与命令优化**
- **使用Pipeline**：将多个无依赖的命令打包成一个请求，减少网络往返次数（RTT）。
- **避免大Key和热Key**：大Key会导致操作阻塞、网络拥堵；热Key会导致单节点压力过大。
- **禁用高危命令**：在生产环境使用`rename-command`禁用`KEYS`, `FLUSHALL`, `FLUSHDB`等可能导致长时间阻塞或数据丢失的命令。
**4. 源码与设计原理角度的根本优化**
- **理解单线程模型**：Redis核心是单线程事件循环（`aeMain`）。任何慢操作都会阻塞整个服务。优化的核心是**避免任何可能阻塞主线程的操作**。
- **I/O多路复用**：Redis使用`epoll`（Linux）等机制，用一个线程管理成千上万的连接，高效处理网络IO。这决定了其高并发能力。
###### 2. Redis 的慢查询日志是什么?
慢查询日志是Redis用于记录执行时间超过预设阈值（`slowlog-log-slower-than`）的命令的日志系统，是性能诊断的关键工具。
**核心特性**
- **记录时机**：记录的是命令在Redis服务端**实际执行时间**，不包括命令排队、网络传输等时间。
- **存储结构**：Redis使用一个**定长的内存列表**（FIFO队列）来存储慢查询日志，其最大长度由`slowlog-max-len`配置。
- **日志内容**：每条记录包含唯一ID、时间戳、命令耗时（微秒）、执行命令及参数，以及客户端信息（4.0+）。
**配置与管理**
```shell
# 动态设置阈值为5毫秒，最大保存1000条记录
CONFIG SET slowlog-log-slower-than 5000
CONFIG SET slowlog-max-len 1000
CONFIG REWRITE  # 持久化到配置文件

# 查看最近的慢查询
SLOWLOG GET 10
# 获取当前慢查询日志数量
SLOWLOG LEN
# 清空慢查询日志
SLOWLOG RESET
[7,8](@ref)
```
**生产建议**：高并发场景下，建议将`slowlog-log-slower-than`设为1ms，`slowlog-max-len`设为1000以上，并定期采集分析。
###### 3. 如何分析 Redis 的性能瓶颈?
**1. 监控系统指标**
- **资源指标**：通过`INFO`命令监控`used_memory`（内存使用）、`connected_clients`（连接数）、`instantaneous_ops_per_sec`（实时QPS）、`used_cpu_sys`/`used_cpu_user`（CPU使用）。
- **持久化指标**：监控`rdb_last_bgsave_status`/`aof_last_bgrewrite_status`（持久化状态）、`aof_delayed_fsync`（AOF阻塞次数）。
**2. 使用慢查询日志**
分析`SLOWLOG GET`的输出，找出执行缓慢的具体命令，这是定位问题最直接的方法。
**3. 使用外部性能工具**
- `redis-benchmark`：进行压力测试，获取Redis实例的极限性能基线。
- `redis-cli --latency`/`--latency-history`：监测客户端到Redis服务器的网络延迟。
- **系统级监控**：使用`top`, `iostat`, `netstat`等工具监控服务器本身的CPU、内存、磁盘IO和网络带宽。
**4. 常见瓶颈根源**
- **CPU/内存**：`INFO`命令查看`used_cpu_sys`等CPU使用指标和`used_memory`等内存指标。
- **网络**：网络带宽打满或延迟过高。
- **命令阻塞**：使用了`KEYS *`、长时间运行的Lua脚本、大Key操作等。
- **持久化fork阻塞**：RDB或AOF重写时，如果内存过大，fork操作会阻塞主进程。
###### 4. Pipeline 是什么?有什么优势?
**Pipeline（管道）**​ 是一种客户端技术，它将多个命令打包后一次性发送给Redis服务器，服务器依次执行所有命令后再将结果一次性返回给客户端。
**工作原理（对比）**
- **普通模式**：`Request1 -> Response1 -> Request2 -> Response2 ...`（N次RTT）
- **Pipeline模式**：`(Request1, Request2, Request3) -> (Response1, Response2, Response3)`（1次RTT）
**优势**
1. **极大减少网络往返时间**：这是最主要优势，在高延迟网络下性能提升极其显著。
2. **提升吞吐量**：单位时间内能处理更多命令。
**Java示例（Jedis）**
```java
Jedis jedis = new Jedis("localhost", 6379);
Pipeline p = jedis.pipelined();
p.set("key1", "value1");
p.get("key1");
p.zadd("sortedSet", 1.0, "member");
Response<String> value = p.get("key1"); // 此时命令未发送，value是Future
p.sync(); // 此时所有命令才被批量发送并执行，结果被填充到各个Response对象
String result = value.get(); // 安全获取结果
```
**注意**：Pipeline中的命令无原子性保证。需要原子性请使用事务（`MULTI/EXEC`）。
###### 5. Redis 的批量操作有哪些?
除了Pipeline，Redis还提供了一些原生批量命令，减少了通信次数。
- `MSET`/`MGET`：批量设置/获取多个String键值对。
- `HMGET`/`HMSET`：批量操作Hash中的多个字段。
- `SADD`/`ZADD`：支持一次添加多个成员到Set或Sorted Set。
- `LPUSH`/`RPUSH`：支持一次插入多个元素到List。
**与Pipeline的选择**
- **原生批量命令**：适用于操作**同一数据结构**的多个字段或值，原子性有保证。
- **Pipeline**：适用于**批量执行不同命令或操作不同Key**的场景，更灵活。
###### 6. 大 key 问题是什么?如何处理?
**大Key**通常指Value的体积过大（如字符串>10KB，哈希/列表/集合/有序集合元素数量>5000）的Key。
**危害**
1. **操作阻塞**：`DEL`大Key、`GET`大字符串会消耗大量CPU，阻塞主线程。
2. **网络拥堵**：传输大Key占用大量带宽，影响其他请求。
3. **内存不均**：在集群中，大Key会导致某个节点内存使用远高于其他节点。
4. **持久化问题**：fork子进程时，内存拷贝压力大，可能导致服务长时间不可用。
**处理方案**
- **拆分**：
    - **数据分片**：将大Hash按字段哈希取模，拆分成多个小Hash，如`user:info:{uid} -> user:info:{uid%10}`。
    - **列表/集合分片**：将一个大的List或Set按元素数量拆分成多个。
- **压缩**：若Value是JSON或文本，可在客户端压缩后存入Redis。
- **使用更合适的数据结构**：例如，用HyperLogLog代替Set进行基数统计。
- **异步删除**：使用`UNLINK`命令（非阻塞）替代`DEL`（阻塞），或启用Lazy Free机制。
###### 7. 热 key 问题是什么?如何处理?
**热Key**是指访问频率远高于其他普通Key的特定Key（如顶流明星的微博信息）。
**危害**
1. **单节点瓶颈**：热Key可能集中在一个Redis实例上，导致该实例CPU、网络负载过高，影响整体服务。
2. **缓存击穿**：热Key过期瞬间，大量请求直接穿透到数据库。
**处理方案**
- **本地缓存**：在应用层使用Guava Cache或Caffeine做二级缓存。需设置合理的过期时间或刷新策略以保证数据一致性。
- **热Key发现与分片**：
    - **监控发现**：通过`redis-cli --hotkeys`（4.0+）或监控分析发现热Key。
    - **逻辑分片**：将热Key复制成多个副本，如`hotkey -> [hotkey:1, hotkey:2, ...]`，客户端访问时随机选择一个副本，将压力分散。
- **Redis集群代理**：使用Twemproxy或Redis Cluster Proxy，它们可在代理层对热Key进行读写分离。
###### 8. Redis 的 Lazy Free 是什么?
Lazy Free（惰性删除）是Redis 4.0引入的特性，旨在解决**删除大Key引发的阻塞问题**。
**原理**
将某些情况下的删除操作从**主线程同步阻塞**改为**使用后台线程异步回收内存**（BIO线程）。
**触发场景**
- **`UNLINK`命令**：与`DEL`不同，`UNLINK`会先逻辑删除Key，后异步回收内存。
- **异步刷新**：使用`FLUSHDB ASYNC`/`FLUSHALL ASYNC`。
- **配置触发**：通过配置`lazyfree-lazy-eviction`、`lazyfree-lazy-expire`等，让内存满淘汰、过期Key删除等场景也启用Lazy Free。
**优势**：避免了删除大Key时导致的服务停顿，提升了稳定性。
###### 9. 如何避免 Redis 的阻塞操作?
**1. 避免长耗时命令**
- 用`SCAN`替代`KEYS`，`SSCAN`/`HSCAN`/`ZSCAN`替代`SMEMBERS`/`HGETALL`等全量遍历命令。
- 复杂排序、交集并集操作在客户端进行。
**2. 控制写操作粒度**
- 避免一次性写入大量数据，如一个包含几万元素的List。
**3. 谨慎使用Lua脚本**
- 确保Lua脚本逻辑轻量，执行时间可控。避免在脚本中执行长时间循环或外部调用。
**4. 关注持久化影响**
- AOF的`appendfsync always`策略每次写入都刷盘，性能最差，通常用`everysec`。
- 控制RDB和AOF重写的触发条件，避免在业务高峰执行。
**5. 使用Lazy Free**
如前所述，使用`UNLINK`和配置Lazy Free来避免大Key删除阻塞。
###### 10. Redis 的网络模型是什么?
Redis的网络模型是其高性能的基石，核心是**单线程Reactor模型**（在Redis 6.0之前）。
**核心组件**
1. **I/O多路复用器**：Redis封装了不同操作系统的I/O多路复用功能（如Linux的`epoll`），使其能够高效地监听成千上万个Socket的读写事件。
2. **事件分发器**：当有Socket事件（如可读、可写）发生时，多路复用器会通知事件分发器。
3. **事件处理器**：事件分发器将事件分发给对应的事件处理器（如连接应答处理器、命令请求处理器、命令回复处理器）。
**工作流程（源码角度简化）**
- **主循环**：在`src/ae.c`的`aeMain`函数中，循环调用`aeProcessEvents`。
- **事件监听**：`aeProcessEvents`调用`epoll_wait`（以epoll为例）等待Socket事件。
- **事件处理**：当有事件到达，根据事件类型调用注册好的处理器。
    - **新连接**：由`acceptTcpHandler`处理，创建客户端状态。
    - **命令请求**：由`readQueryFromClient`处理，读取命令、解析并放入队列，最后由`processCommand`执行命令。
    - **结果回复**：命令执行完后，将回复数据写入客户端缓冲区，并监听可写事件，由`sendReplyToClient`将数据发送给客户端。
**Redis 6.0的演进**
Redis 6.0引入了**多线程I/O**，但**命令执行仍然是单线程**。多线程只用于处理**网络数据的读取和解析**（`readQueryFromClient`）以及**回复数据的序列化和发送**（`sendReplyToClient`）。这缓解了网络IO成为瓶颈的情况，核心的命令执行逻辑依然保持了单线程的简单性和原子性。