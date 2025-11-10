### 一、基础概念类
###### 1. ZooKeeper 是什么?
###### 2. ZooKeeper 能做什么?
###### 3. 你说说 Zookeeper 文件系统
###### 4. Zookeeper 的系统架构又是怎么样的?
###### 5. 说说 zookeeper 有哪些数据节点
###### 6. Znode 里面都存储了什么?
###### 7. Zookeeper 都有哪些功能?
###### 8. Zookeeper 的 ACL 权限控制机制是什么?
###### 9. 什么是 ZNode 的版本号(version)?
###### 10. Zookeeper 的数据模型是什么样的?
### 二、Leader 选举与集群管理类
###### 1. Zookeeper 初始化是如何进行 Leader 选举的
###### 2. 如果 Leader 挂了,进入崩溃恢复,怎么选举 Leader?
###### 3. 选举 leader 后是怎么进行数据同步的
###### 4. Zookeeper 选举核心参数有哪些
###### 5. 分布式集群中为什么会有 Master 主节点?
###### 6. zk 节点宕机如何处理?
###### 7. 服务器有哪些角色
###### 8. Zookeeper 下 Server 工作状态
###### 9. Zookeeper server 有哪些状态?
###### 10. Leader 选举的 FastLeaderElection 算法原理是什么?
###### 11. 什么是 myid 和 epoch?
###### 12. Observer 角色的作用是什么?
### 三、数据同步与一致性类
###### 1. Zookeeper 怎么保证主从节点的状态同步?
###### 2. 能说说数据是如何同步的吗?
###### 3. Zookeeper 是如何保证事务的顺序一致性的?
###### 4. 那你详细给我讲讲 ZAB 协议吧
###### 5. ZAB 和 Paxos 算法的联系与区别?
###### 6. 为什么说 Zookeeper 不是 AP 模型?
###### 7. Zookeeper 如何保证数据的最终一致性?
###### 8. 什么是 ZXID(ZooKeeper Transaction ID)?
###### 9. Zookeeper 的两阶段提交(2PC)是如何实现的?
###### 10. ZAB 协议的崩溃恢复和消息广播阶段分别是什么?
###### 11. Zookeeper 如何处理脑裂问题?
### 四、Watcher 监听机制类
###### 1. 说说 Zookeeper Watcher 机制
###### 2. 客户端是如何注册 Watcher 实现
###### 3. 服务端是如何处理 Watcher 实现
###### 4. 客户端是如何回调 Watcher
###### 5. Zookeeper 对节点的 watch 监听通知是永久的吗?
###### 6. 说一下 Zookeeper 的通知机制?
###### 7. Watcher 的特性有哪些?
###### 8. Watcher 的触发类型有哪些?
###### 9. Watcher 的注册和触发流程是怎样的?
### 五、会话管理类
###### 1. 熟悉会话管理吗
###### 2. 了解 Chroot 特性吗
###### 3. Zookeeper 的会话超时机制是什么?
###### 4. 什么是 Session ID 和 Session Password?
###### 5. Zookeeper 客户端与服务端的心跳机制是怎样的?
###### 6. 会话的生命周期包含哪些状态?
###### 7. Zookeeper 如何处理会话迁移?
### 六、部署与集群配置类
###### 1. Zookeeper 有哪几种几种部署模式?
###### 2. 集群最少要几台机器,集群规则是怎样的?集群中有 3 台服务器,其中一个节点宕机,这个时候 Zookeeper 还可以使用吗?
###### 3. 集群支持动态添加机器吗?
###### 4. Zookeeper 负载均衡和 nginx 负载均衡区别
###### 5. Zookeeper 的配置文件(zoo.cfg)中重要参数有哪些?
###### 6. Zookeeper 如何进行扩容和缩容?
###### 7. 什么是 Zookeeper 的 Quorum(法定人数)?
###### 8. 为什么 Zookeeper 集群建议使用奇数台服务器?
### 七、客户端与 API 类
###### 1. Zookeeper 的 java 客户端都有哪些?
###### 2. Curator 框架的优势是什么?
###### 3. Zookeeper 原生客户端的缺点有哪些?
###### 4. Zookeeper 客户端常用的操作有哪些?
###### 5. 如何使用 Zookeeper 实现分布式锁?
###### 6. Zookeeper 如何实现分布式队列?
### 八、应用场景类
###### 1. Zookeeper 的典型应用场景
###### 2. Zookeeper 和 Dubbo 的关系?
###### 3. 如何使用 Zookeeper 实现服务注册与发现?
###### 4. 如何使用 Zookeeper 实现配置中心?
###### 5. 如何使用 Zookeeper 实现命名服务?
###### 6. 如何使用 Zookeeper 实现集群管理?
###### 7. 如何使用 Zookeeper 实现 Master 选举?
###### 8. Zookeeper 在 Kafka 中的应用是什么?
###### 9. Zookeeper 在 HBase 中的应用是什么?
### 九、对比与选型类
###### 1. Zookeeper 和 Eureka、Consul、Nacos 有什么区别?
###### 2. chubby 是什么,和 zookeeper 比你怎么看?
###### 3. Zookeeper 和 etcd 的区别是什么?
###### 4. 在什么场景下应该选择 Zookeeper?
###### 5. Zookeeper 的 CP 特性适合什么场景?
###### 6. Zookeeper 与 Redis 分布式锁的区别?
### 十、性能与优化类
###### 1. Zookeeper 的性能瓶颈在哪里?
###### 2. 如何优化 Zookeeper 的性能?
###### 3. Zookeeper 的写性能为什么不高?
###### 4. Zookeeper 的事务日志和快照机制是什么?
###### 5. 如何进行 Zookeeper 的数据清理?
###### 6. Zookeeper 的内存管理机制是怎样的?
###### 7. 什么情况下会导致 Zookeeper 性能下降?
### 十一、安全与运维类
###### 1. Zookeeper 如何保证数据安全?
###### 2. Zookeeper 的身份认证机制有哪些?
###### 3. 如何监控 Zookeeper 集群的健康状态?
###### 4. Zookeeper 的四字命令有哪些?
###### 5. 如何进行 Zookeeper 的数据备份与恢复?
###### 6. Zookeeper 的日志如何管理?
###### 7. 如何排查 Zookeeper 的常见问题?
###### 8. Zookeeper 集群升级需要注意什么?
### 十二、故障处理与问题排查类
###### 1. Zookeeper 连接超时如何处理?
###### 2. 如何处理 Zookeeper 的 Session Expired?
###### 3. Zookeeper 数据不一致如何排查?
###### 4. 如何解决 Zookeeper 的羊群效应(Herd Effect)?
###### 5. 什么是 Zookeeper 的惊群问题?
###### 6. Zookeeper 网络分区如何处理?
###### 7. Zookeeper 磁盘空间不足如何处理?