### 一、Kafka 基础概念

###### 1. Kafka 是什么？有什么作用？
###### 2. Kafka 的核心组件有哪些？
###### 3. 什么是 Topic、Partition 和 Offset？
###### 4. Kafka 中的 Broker 是什么？
###### 5. 什么是消费者组（Consumer Group）？
###### 6. Kafka 中的 Replication 是什么？
###### 7. 什么是 Leader 和 Follower？
###### 8. Kafka 的消息模型是什么？

### 二、Kafka 架构与设计

###### 1. Kafka 的整体架构是怎样的？
###### 2. Kafka 如何实现分布式？
###### 3. Kafka 的存储机制是什么？
###### 4. Kafka 的日志段（Log Segment）是什么？
###### 5. Kafka 如何实现高吞吐量？
###### 6. Kafka 的零拷贝（Zero Copy）是什么？
###### 7. Kafka 的页缓存（Page Cache）机制是什么？
###### 8. Kafka 的顺序写入是如何实现的？

### 三、生产者相关

###### 1. Kafka 生产者的工作流程是什么？
###### 2. 生产者如何选择将消息发送到哪个分区？
###### 3. 什么是生产者的分区策略？
###### 4. 生产者的 acks 参数有哪些值？分别代表什么？
###### 5. 如何保证生产者发送消息的可靠性？
###### 6. 生产者的幂等性是什么？如何实现？
###### 7. 生产者的事务是什么？如何使用？
###### 8. 什么是生产者的批量发送？
###### 9. 生产者的 buffer.memory 参数是什么？
###### 10. 生产者的 linger.ms 和 batch.size 参数是什么？
###### 11. 生产者的压缩机制是什么？
###### 12. 生产者发送消息失败时会发生什么？
###### 13. 什么是生产者的重试机制？
###### 14. 生产者缓冲区满了怎么办？

### 四、消费者相关

###### 1. Kafka 消费者的工作流程是什么？
###### 2. 消费者是如何订阅 Topic 的？
###### 3. 什么是消费者的 Rebalance？
###### 4. Rebalance 会带来什么问题？
###### 5. 如何避免不必要的 Rebalance？
###### 6. 消费者的 Offset 是如何管理的？
###### 7. 消费者的自动提交和手动提交有什么区别？
###### 8. 如何保证消费者消费消息的可靠性？
###### 9. 消费者如何实现精确一次消费（Exactly Once）？
###### 10. 消费者的拉取模式（Pull）有什么优势？
###### 11. 消费者如何处理消费延迟？
###### 12. 什么是消费者的心跳机制？
###### 13. 消费者的 poll() 方法是如何工作的？
###### 14. 如何实现消费者的多线程消费？
###### 15. 消费者的分区分配策略有哪些？

### 五、Kafka 高可用与容错

###### 1. Kafka 如何保证高可用性？
###### 2. 什么是 ISR（In-Sync Replicas）？
###### 3. Leader 选举机制是什么？
###### 4. 什么是 Unclean Leader 选举？
###### 5. Kafka 如何处理 Broker 宕机？
###### 6. 什么是 Controller 节点？它的作用是什么？
###### 7. Controller 如何选举？
###### 8. Kafka 的副本同步机制是什么？
###### 9. 什么是 HW（High Watermark）和 LEO（Log End Offset）？
###### 10. Kafka 如何保证数据不丢失？
###### 11. Kafka 如何保证数据不重复？
###### 12. 什么是 Kafka 的最少同步副本数（min.insync.replicas）？

### 六、Kafka 性能优化

###### 1. 如何提高 Kafka 的吞吐量？
###### 2. 如何降低 Kafka 的延迟？
###### 3. Kafka 的分区数应该如何设置？
###### 4. Kafka 的副本数应该如何设置？
###### 5. 如何优化 Kafka 的磁盘 I/O？
###### 6. 如何优化 Kafka 的网络传输？
###### 7. Kafka 的消息压缩如何选择？
###### 8. 如何监控 Kafka 的性能指标？
###### 9. Kafka 的日志清理策略有哪些？
###### 10. 如何设置 Kafka 的数据保留时间？

### 七、Kafka 集群管理

###### 1. 为什么要使用 Kafka 集群？
###### 2. 如何搭建 Kafka 集群？
###### 3. Kafka 集群如何扩容？
###### 4. Kafka 集群如何缩容？
###### 5. 如何进行 Kafka 集群的升级？
###### 6. 什么是 Kafka 的地域复制？
###### 7. 什么是 MirrorMaker？它的作用是什么？
###### 8. MirrorMaker 和 MirrorMaker 2 有什么区别？
###### 9. Kafka 集群如何进行跨机房部署？
###### 10. 如何备份和恢复 Kafka 数据？

### 八、Kafka 安全性

###### 1. Kafka 支持哪些认证机制？
###### 2. 什么是 Kafka 的 SASL 认证？
###### 3. Kafka 如何配置 SSL 加密？
###### 4. Kafka 的 ACL（访问控制列表）是什么？
###### 5. 如何配置 Kafka 的权限管理？

### 九、Kafka 与其他组件对比

###### 1. Kafka 和 RabbitMQ 的区别是什么？
###### 2. Kafka 和 RocketMQ 的区别是什么？
###### 3. Kafka 和 Pulsar 的区别是什么？
###### 4. Kafka 和 Flume 的主要区别是什么？
###### 5. Kafka 和 ActiveMQ 的区别是什么？

### 十、Kafka 实战应用

###### 1. Kafka 在日志收集中的应用场景？
###### 2. Kafka 在实时数据处理中的应用场景？
###### 3. Kafka 如何与 Spark Streaming 集成？
###### 4. Kafka 如何与 Flink 集成？
###### 5. Kafka 如何与 Storm 集成？
###### 6. Kafka 如何与 Elasticsearch 集成？
###### 7. Kafka 在微服务架构中的应用？
###### 8. Kafka 在 CQRS 架构中的应用？
###### 9. Kafka 在事件溯源（Event Sourcing）中的应用？

### 十一、Kafka Streams

###### 1. 什么是 Kafka Streams？
###### 2. Kafka Streams 的核心概念有哪些？
###### 3. Kafka Streams 的拓扑（Topology）是什么？
###### 4. Kafka Streams 的状态存储是什么？
###### 5. Kafka Streams 如何实现窗口操作？
###### 6. Kafka Streams 如何实现 Join 操作？
###### 7. Kafka Streams 和 Spark Streaming 的区别？

### 十二、Kafka Connect

###### 1. 什么是 Kafka Connect？
###### 2. Kafka Connect 的核心概念有哪些？
###### 3. 什么是 Source Connector 和 Sink Connector？
###### 4. 如何开发自定义的 Kafka Connector？
###### 5. Kafka Connect 的分布式模式和独立模式有什么区别？

### 十三、Kafka 问题排查

###### 1. 如何排查 Kafka 消息丢失问题？
###### 2. 如何排查 Kafka 消息重复问题？
###### 3. 如何排查 Kafka 消费延迟问题？
###### 4. 如何排查 Kafka Rebalance 频繁问题？
###### 5. 如何排查 Kafka 生产者发送慢的问题？
###### 6. 如何排查 Kafka Broker 宕机问题？
###### 7. 如何排查 Kafka 磁盘空间不足问题？