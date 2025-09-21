# 携程异地多活-MySQL 实时双向（多向）复制

# 实践

# 一、前言

携程内部 MySQL 部署采用多机房部署，机房 A 部署一主一从，机房 B 部署一从，作为 DR（Disaster
Recovery）切换使用。当前部署下，机房 B 部署的应用需要跨机房进行写操作；当机房 A 出现故障时，
DBA 需要手动对数据库进行 DR 切换。

为了做到真正的数据异地多活，实现 MySQL 同机房就近读写，机房故障时无需进行数据库 DR 操作，只
进行流量切换，就需要引入数据实时双向（多向）复制组件。

## 二、DRC 介绍

DRC（Data Replicate Center）是携程框架架构研发部推出的用于数据双向或多向复制的数据库中间
件，在公司 G 2（高品质 Great Service、全球化 Globalization）战略的背景下，服务于异地多活项目，
赋予了业务全球化的部署能力。

## 三、DRC 架构设计

DRC 采用服务端集中化设计，配合另一数据库访问中间件 DAL（Data Access Layer）的本地读写功能，
实现数据就近访问。


#### 模块介绍

```
Replicator Container
```
Replicator Container 实现对 Replicator 实例的管理，一个 Replicator 实例表示对一个 MySQL 集群的
复制单元，Instance 将自己伪装为 MySQL 的 Slave，实现 Binlog 的拉取和本地存储。

```
Applier Container
```
Applier Container 实现对 Applier 实例的管理，一个 Applier 实例连接到一个 Replicator 实例，实现对
Replicator 实例本地存储 Binlog 的拉取，进而解析出 SQL 语句并应用到目标 MySQL，从而实现数据的复
制。

```
Cluster Manager
```
Cluster Manager 负责集群高可用切换，包括由于 MySQL 主从切换导致的 Replicator 实例和 Applier 实
例重启，以及 Replicator 实例与 Applier 实例自身主从切换引起的新实例启动通知。

```
Console
```
Console 提供 UI 操作、外部系统交互 API 以及监控告警。

## 四、DRC 详细设计

## 4.1 接入 DB 规范

#### DRC 的核心指标包括复制延迟和数据一致性。

为了实现数据复制的低延迟，Applier 能够快速应用 SQL，就需要每个表至少包含主键或者唯一键，加速
执行效率；同时在保证数据准确的前提下，SQL 应该尽量并行复制，需要 MySQL 开启从 5.7.22 版本引入
的 Writeset 功能。

为了保证数据复制的准确性，在主备切换时 Replicator 仍能准确定位 Binlog 位点，需要 MySQL 开启
GTID；当数据复制发生冲突时，为了具备自动解决冲突的能力，需要表包含时间戳列，并精确到毫秒。

这就需要接入 DRC 的 MySQL 数据库满足：

1 ）5.7.22 及以上版本；

2 ）Master 上开启 Writeset 并行复制；

3 ）MySQL 开启 GTID；

4 ）每个表包含时间戳列，精确到毫秒；

5 ）每个表至少包含主键或者唯一键。


DRC 的复制依赖 GTID（Global Transaction ID），这里先简单介绍一下 GTID 的概念。MySQL 5.6.5 版本
新增了一种基于 GTID 的复制方式，强化了数据库的主备一致性，故障恢复以及容错能力，取代传统的基
于 file 和 position 主从复制，使得在 MySQL 主备切换时，仍能准确定位到 Binlog 位点。

GTID 的格式形如：source_id: transaction_id，其中 source_id 表示 MySQL 服务器的 uuid，
transaction_id 是在事务提交的时候系统顺序分配的一个序列号。

## 4.2 Binlog 复制

单向复制链路包含拉取 Binlog 并持久化到本地磁盘的 Replicator，和请求 Binlog 且并行应用到目标
MySQL 的 Applier。整个链路涉及的 I/O 操作包括网络传输和磁盘读写。

### 4.2.1 低复制延迟

#### 为了降低复制延迟，就要求复制链路中每一环都尽可能高效。网络层通信模型使用异步 I/O；系统层尽

可能使用操作系统提供的 **Zero Copy** 和 **Page Cache** ；应用层提高数据处理并行度以及降低系统不可用
时间。

监控显示生产环境业务双向复制延迟 999 线 < 1 s。下面就介绍一下 DRC 在降低复制延迟方面所做的性能
优化工作。

### 1 ）网络层


Replicator 采用 GTID 复制方式，实现了 MySQL 复制协议，伪装成源 MySQL 的 Slave 拉取 Binlog。网络层
通信组件采用携程开源组件 XPipe (https://github.com/ctripcorp/x-pipe)，实现网络交互异步化。

### 2 ）系统层

接收 Binlog 时，从数据流中解析出不同类型的 Event，直接保存在 **堆外内存** 。

每个 Event 需要经过一组过滤器，进而决定是否需要落盘持久化。

```
对于Heartbeat类型的Event需要过滤丢弃；针对某些不需要进行数据同步的库和表，需要丢弃相
应Event，减少存储量和传输量；
```
**高性能 IO 的要点**

```
对于需要持久化的Event，直接将堆外内存中的数据写入文件Page Cache并定时刷入磁盘，减少
数据复制和IO操作，降低处理耗时，提升Replicator拉取效率。
```
发送 Binlog 时，当 Applier 进度落后 Replicator，需要从磁盘读取，这时只解析 gtid_event 事件，其他需
要发送的事件直接从磁盘读取到堆外内存进行发送，减少数据复制。

### 3 ）应用层

Applier 借鉴原生 MySQL 基于 Writeset 的并行复制，内嵌了基于水位的并行算法，高效的将 SQL 应用到
目标数据库。

除去正常复制之外，为了降低系统的不可用时间，就需要系统在异常情况下，尽快恢复正常功能。比如
断网恢复时，为了避免一端使用老连接，就需要对连接进行空闲检测；为了应对断网导致数据堆积出现
流量突增，就需要对流量进行控制。

### 4 ）空闲检测

**Replicator 与 MySQL、Applier 和 Replicator 通过 Netty 进行数据传输** ，当网络出现故障，可能一端
仍然使用老连接进行通信，会导致数据复制出现中断。

针对网络故障，Replicator 对 MySQL 添加了读空闲检测，启动时设置 MySQL 空闲时间隔 10 s 发送一次
heartbeat_event，如果 30 s 没有收到 MySQL 任何事件，则认为 MySQL 出现问题，发起重连。

Replicator 对 Applier 设置了写空闲检测，当没有 Event 需要发送给 Applier 时，间隔 10 s 发送一次
heartbeat_event，如果发送失败，则认为 Applier 出现问题，断开连接。

Applier 对 Replicator 设置了读空闲检测，如果 30 s 没有收到 Replicator 任何事件，则认为 Replicator 出现
问题，发起重连。

### 5 ）流量控制

设计上 Replicator Container 使用物理机，其中会运行若干 Replicator 实例，Applier Container 使用虚
拟机，这样会造成发送和消费的速率不匹配。

尤其当 Applier 由于某种原因出现故障后，在 Replicator 端堆积大量未消费的 Event，重启后如果堆积的
Event 全部发送过来，可能会直接打垮 Applier，这样就需要在 Replicator 实例上对 Applier 进行限流。

Replicator 发送端使用 Netty 提供的 **WRITE_BUFFER_WATER_MARK 高低水位的变化来控制流控的开
关** ，进而动态调整发送速率，整形平滑流量。

### 4.2.2 数据一致性

#### 为了保证数据的一致，就需要满足：


#### 1 ）数据拉取时保证时序；

2 ）数据拉取不能遗漏，SQL 应用时不重，或者即使重复，要保证幂等操作，保证 At Least Once；

3 ）数据冲突时，能正确处理，保证数据最终一致。

下面就看下 DRC 是如何保证以上 3 个要求。

### 1 ）时序保证

本地磁盘保存 Binlog 采用原生的存储协议，Replicator 顺序处理接收到每一个 Event 事件。

存储协议兼容 MySQL 原生的 mysqlbinlog 命令，其中根据 DRC 自身的需要，保存了自定义的一些辅助事
件，比如 DDL 事件，表结构事件。消费时顺序发送 Binlog 文件中的事件给 Applier。

**2 ）At Least Once**

为了实现 At Least Once，需要解决 3 个子问题：

1 ）Replicator 或者 Applier 重启时，如何保证请求的 GTID set 准确体现目前的消费偏移？

2 ）双向（多向）复制如何解决循环复制？

3 ）Applier 由于异常重复拉取时，如何保证幂等？

下面逐一介绍每个子问题的解决方案。

### 断点重续

当 Replicator 重启时，会从本地磁盘中恢复已经拉取过的 GTID set：

1 ）定位重启前使用的最后一个 Binlog 文件；

2 ）解析出 previous_gtids_event；

3 ）遍历该文件的所有 gtid_event，与 previous_gtids_event 解析出的 GTID set 取并集。

恢复过程中，会校验文件的正确性，对于没有以 xid_event 结束的事务，Replicator 会对文件进行截断，
对应的 gtid 事务会重新请求。

当 Applier 重启时，Cluster Manager 会从目标数据库中查询出当前已经执行过的 GTID set 发送给
Applier，Applier 带着该参数向 Replicator 发送 Binlog 拉取请求。Replicator 收到请求中的 GTID set，从
本地磁盘中定位出第一个需要发送的 Event 所在的 Binlog 文件，依次遍历该文件中的每一个 Event，针对
gtid_event 事件取出其中的 gtid，判断该 gtid 对应的事务是否包含在 GTID set 中，如果包含其中，则表示
Applier 已经消费过，无需发送，否则通过堆外内存直接将 Event 发送给 Applier。


### 循环复制

单向复制时，经过 DRC 复制到对端的 SQL 在执行后，同样会落到 MySQL 的 Binlog 中，这样在双向 (多向)
复制结构中，对端的 Replicator Instance 在拉取到该条 Binlog 后如果继续复制，就会出现循环复制的问
题。

针对循环复制，业内可选的解决方案是在 Binlog 事务开头插入一条写操作，标识出该条事务是 DRC 复制
过来，而不是真实业务写入，这样对端 Replicator 发现一个事务开头包含 DRC 特殊标记时，就不会继续
复制该事务。

分析 MySQL 自身主从复制，Slave 在收到 Master 同步过来的 Binlog 时，通过 set gtid_next 将该事务的
GTID 设置为同步过来的 gtid_event 中的 GTID，这样就实现了主从 GTID set 的一致性。

如果将 Replicator 拉取 Binlog 类比为 Slave 的 I/O 线程，磁盘文件类比为 Relay log，Applier 类比为 Slave
的 SQL 线程，那么 Applier 是可以采用同样的方式，使用 set gtid_next 设置经过 DRC 复制到对端事务的
GTID，这样源和目标数据库的 GTID set 会保持一致，更重要的是可以标识出该事务是经 DRC 复制过来
的。这也是 DRC 最终采用的破解循环复制的方案。

如下双向复制结构，Replicator Instance 1 只会同步源 MySQL 集群 uuidSet 1 中的服务器产生事务，
Replicator Instance 2 只会同步目标 MySQL 集群 uuidSet 2 中的服务器产生事务。如果业务在源 MySQL 集
群写入一条数据，Replicator Instance 1 从 gtid_event 中的 GTID 解析出 uuid 属于 uuidSet 1，那么会持久
化到磁盘并发送给 Applier Instance 1，Applier Instance 1 接收到事务中包含的所有 Event 后，执行 set
gtid_next=GTID，然后通过 JDBC 将 SQL 写入目标 MySQL，完成单向复制；Replicator Instance 2 接收到
gtid_event 后，同样解析出 GTID，但是 uuid 并不属于 uuidSet 2，这样该条事务就会被过滤，从而避免的
循环复制。

### 幂等

Applier 如果重复接收到相同 GTID 的事务，由于 MySQL 会记录已经执行的 GTID set，如果该 GTID 已经被
执行，则会自动忽略，这样即使 Applier 重复应用同一条事务，也不会对业务产生影响。

### 小结

从上面可以看到，在保证数据一致性时，GTID 不论是在 Replicator 和 Applier 重启后 Binlog 位点定位，标
识 Binlog 来源避免循环复制，还是 Applier 重复应用时幂等实现，都起到了至关重要的作用。

### 3 ）冲突解决

#### 设计上，首先要避免冲突的出现：

1 ）接入 Set 化的业务在流量入口处就会根据 uid 进行分流，同一个用户的流量进入同一个机房；数据接
入层中间件 DAL 同样会采用 local-2-local 的路由策略。这样同一条记录在 2 个机房同时被修改的情况很少
发生；


#### 2 ）对于使用自增 ID 的业务，通过不同机房设置不同的自增 ID 规则，或者采用分布式全局 ID 生成方案，

#### 避免双向复制后数据冲突。

#### 如果数据确实出现了冲突， 2 个机房对同一条数据进行的修改，这时需要根据冲突处理策略进行处理：

1 ）Applier 根据默认的冲突处理策略进行处理，接入 DRC 的表都有一个精确到毫秒自动更新的时间戳，
冲突时时间戳靠后的会被采用，进而实现数据的一致；

2 ）冲突的 SQL 会被监控记录，连同数据库中的原始数据同时提供给用户，进而自助决定是否需要进行
覆盖。

## 4.3 DDL 支持

DDL 操作会引起表结构的变更，在复制链路中 Applier 需要表结构信息解析对应时刻的 Binlog Event，当
Applier 消费速率落后 Replicator 的发送速率时，就需要历史版本的表结构信息才能够正确解析 Binlog
Event。

这就引入了表结构设计第一个问题：历史版本如何存储？

为了存储表结构，势必首先要获得表结构，如果从源 MySQL 直接抓取表结构，由于 Binlog 是异步发送，
就导致抓取到 DDL 的 Binlog 时刻，与 MySQL 上表结构未必能够一一对应，从而引起 Applier 解析出现问
题，进而导致数据不一致。这就引入表结构设计第二个问题：表结构从何处抓取？

业界通用的解决方案是基于独立的第 3 方数据库进行表结构单独存储管理。数据库本身就是存储工具，
Snapshot 表和 DDL 表分别保存表结构快照和 DDL 变更记录，这样任意时刻的表结构等于 Snapshot 及其
后 DDL 变更集合，则第一个表结构存储问题顺其自然得以解决；独立数据库镜像一份源数据库的库表结
构，每次从 Binlog 接收到 DDL Event 后，将解析出的 DDL 语句直接应用到镜像数据库，随即抓取相应表
结构即可，这样就解决了第二个表结构从何处抓取的问题。

#### 独立数据库解决方案的缺点是引入外部依赖，降低了系统的可用性，提高了运维成本。

### 4.3.1 表结构存储和计算

#### 针对 DDL 功能中问题一：

从数据库中查询 Snapshot 和 DDL 记录的好处是时间顺序容易确定，能够简单准确的恢复表结构。那么
是否有其他存储介质，在保存表结构快照和 DDL 操作的同时，能够保证时序呢？有，保存 Binlog 的文件
就具有这种特性，DRC 采用了这种基于 Binlog 的表结构文件存储方案。


#### 针对 DDL 功能中问题二：

#### 镜像数据库是为了实时计算出 DDL 变更后最新的表结构信息，在存储不使用独立部署的数据库后，DRC

#### 引入嵌入式轻量数据库，降低外部依赖和系统运维成本。

#### 这样整体的设计方案如下图所示：

Binlog 文件头会保存自定义表结构快照事件，当从接收的 Event 事件检测到 DDL 后，保存为自定义的
DDL 事件。这样当 Applier 连接上 Replicator 后，总是会根据 GTID set 定位到需要的第一个历史版本表结
构所在的文件，从而实时恢复表结构历史，用于后续 Binlog Event 的解析。

我们将数据库最小依赖打成独立的 Jar 包服务，每个 Replicator 实例启动时，会一并启动一个独立的嵌入
式数据库，在恢复 GTID set 的同时，根据表结构快照事件和 DDL 事件重建嵌入式数据库中表结构。

### 4.3.2 DDL 入口

携程内部发布 DDL 是通过 gh-ost 进行变更，gh-ost 会在影子表中执行 DDL 操作，等影子表中数据同步完
成后，业务低峰期进行原表和影子表的切换。

针对 gh-ost，需要追踪 gh-ost 变更过程中内部形如_xxx_gho 的表的 DDL 所有操作，最终执行切换时检测
出 rename 操作，保存对应表结构最新信息发送给 Applier 即可。

同时针对数据库直接进行的 DDL 操作，直接检测出 DDL 类型的 Event 即可。

### 4.3.3 DDL 异常处理

#### 对于接入 DRC 的数据库，当在进行 DDL 变更时，可能会出现两边数据库变更不同步，单侧进行了 DDL 变

更，另一侧未进行变更。针对新增列这种场景，Applier 在保证数据一致的前提下，对新增列的值进行
比较，如果 Binlog 中解析出的值和该列的默认值一致，则会剔除该列，继续数据复制。这样在另一侧补
上 DDL 变更后，两侧的数据最终仍然一致。

## 4.4 监控告警

#### DRC 核心指标包括复制延迟和数据一致性。除此之外我们还提供 BU、应用和 IDC 维度的监控：

#### 1 ）流量和 TPS 监控告警；

#### 2 ）BU、应用和 IDC 维度的监控告警；

#### 3 ）DDL 变更监控；


#### 4 ）表结构一致性监控告警；

#### 5 ）数据冲突监控；

6 ）GTID set GAP 监控。

## 五、总结

#### 本次分享围绕 DRC 的核心指标复制延迟和数据一致性，介绍了复制过程中对性能的优化以及各种场景如

何保证数据的一致性。针对 DDL，分别支持 gh-ost 和直接 DDL 操作，实现在线表结构变更不影响数据复
制。

后续 DRC 的工作会集中在高可用、海外支持上以及外围设施的建设上，为携程的国际化战略提供数据层
面的支撑。

**【作者简介】** Roy，携程软件技术专家，负责 MySQL 双向同步 DRC 和数据库访问中间件 DAL 的开发演
进，对分布式系统高可用设计、数据一致性领域感兴趣。


