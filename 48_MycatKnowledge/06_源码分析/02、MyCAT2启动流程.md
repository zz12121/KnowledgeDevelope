# MyCAT2 启动流程源码分析

## 1. 启动入口

### 1.1 MycatCore 类

```java
public class MycatCore {
    // 以 vertx 方式启动 mycat2
    public void startServer() throws Exception {
        // 加载配置文件
        ConfigUpdater.loadConfigFromFile();
        
        // 默认 Vertx 方式启动
        mycatServer.start();
        
        // 普罗米修斯指标导出
        new PrometheusExporter().run();
    }
}
```

---

## 2. 配置文件加载

### 2.1 ConfigUpdater 类

```java
public class ConfigUpdater {
    @SneakyThrows
    public static void load(MycatRouterConfig mycatRouterConfig) {
        StorageManager storageManager = MetaClusterCurrent.wrapper(StorageManager.class);
        MycatRouterConfig orginal = MetaClusterCurrent.exist(MycatRouterConfig.class) ?
            MetaClusterCurrent.wrapper(MycatRouterConfig.class) : new MycatRouterConfig();
        
        MycatRouterConfigOps mycatRouterConfigOps = new MycatRouterConfigOps(
            orginal, 
            storageManager, 
            Options.builder().persistence(false).build(), 
            mycatRouterConfig
        );
        
        mycatRouterConfigOps.commit();
    }
}
```

### 2.2 配置提交流程

```java
public void commit() throws Exception {
    ServerConfig serverConfig = MetaClusterCurrent.wrapper(ServerConfig.class);
    boolean init = isInit();
    MycatRouterConfig newConfig = this.newConfig;
    
    // 检查配置是否变更
    if (!init && Objects.equals(this.original, newConfig)) {
        LOGGER.info("config no changed");
        return;
    }
    
    // 创建配置更新集合
    UpdateSet<LogicSchemaConfig> schemaConfigUpdateSet = 
        UpdateSet.create(newConfig.getSchemas(), original.getSchemas());
    UpdateSet<ClusterConfig> clusterConfigUpdateSet = 
        UpdateSet.create(newConfig.getClusters(), original.getClusters());
    UpdateSet<DatasourceConfig> datasourceConfigUpdateSet = 
        UpdateSet.create(newConfig.getDatasources(), original.getDatasources());
    UpdateSet<SequenceConfig> sequenceConfigUpdateSet = 
        UpdateSet.create(newConfig.getSequences(), original.getSequences());
    // ... 更多配置更新
}
```

---

## 3. 核心组件初始化

### 3.1 组件加载顺序

```java
// 1. 停止旧的副本选择器管理器
if (MetaClusterCurrent.exist(ReplicaSelectorManager.class)) {
    ReplicaSelectorManager replicaSelectorManager = 
        MetaClusterCurrent.wrapper(ReplicaSelectorManager.class);
    replicaSelectorManager.stop();
}

// 2. 获取 JDBC 连接管理器
Resource<JdbcConnectionManager> jdbcConnectionManager = 
    getJdbcConnectionManager(datasourceConfigUpdateSet);

// 3. 获取数据源配置提供者
Resource<DatasourceConfigProvider> datasourceConfigProvider = 
    getDatasourceConfigProvider(datasourceConfigUpdateSet);

// 4. 获取 MySQL 管理器
Resource<MySQLManager> mycatMySQLManager = 
    getMycatMySQLManager(datasourceConfigUpdateSet);

// 5. 获取副本选择器管理器
Resource<ReplicaSelectorManager> replicaSelectorManager = 
    getReplicaSelectorManager(clusterConfigUpdateSet, 
        datasourceConfigUpdateSet, 
        jdbcConnectionManager);

// 6. 注册到元数据集群
MetaClusterCurrent.register(JdbcConnectionManager.class, jdbcConnectionManager.get());
MetaClusterCurrent.register(ConnectionManager.class, jdbcConnectionManager.get());
MetaClusterCurrent.register(DatasourceConfigProvider.class, datasourceConfigProvider.get());
MetaClusterCurrent.register(ReplicaSelectorManager.class, replicaSelectorManager.get());
MetaClusterCurrent.register(MySQLManager.class, mycatMySQLManager.get());
```

---

## 4. Vertx 启动 MyCAT2

### 4.1 VertxHandler 加载

MyCAT2 使用 Vertx 作为网络通信框架，主要流程：

1. **创建 Vertx 实例**
2. **加载 MySQL 协议处理器**
3. **绑定端口监听**

### 4.2 客户端监听入口

```java
// VertxMySQLPacketResolver
// 使用 epoll 模型处理客户端连接
```

---

## 5. SQL 执行流程

### 5.1 核心处理类

| 类名 | 作用 |
|------|------|
| MycatVertxMySQLHandler | 处理 MySQL 协议请求 |
| MycatdbCommand | 执行计划管理 |
| ShardingSQLHandler | 核心查询处理 |
| ResultSetHandler | 结果归并处理 |

### 5.2 执行流程

```
客户端请求
    ↓
MycatVertxMySQLHandler 接收请求
    ↓
MycatdbCommand 执行计划管理
    ↓
ShardingSQLHandler 分片路由
    ↓
ResultSetHandler 结果归并
    ↓
返回结果给客户端
```

---

## 6. 源码流程图

```
┌─────────────────────────────────────────────────────────────┐
│                        启动入口                              │
│                   MycatCore.startServer()                  │
└─────────────────────────────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────┐
│                   加载配置文件                               │
│              ConfigUpdater.loadConfigFromFile()            │
└─────────────────────────────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────┐
│              初始化核心组件                                  │
│  JdbcConnectionManager / ReplicaSelectorManager           │
│  DatasourceConfigProvider / MySQLManager                   │
└─────────────────────────────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────┐
│              Vertx 启动服务                                  │
│         加载 Handler，绑定端口 8066                         │
└─────────────────────────────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────┐
│                    服务运行中                                │
│      接收 SQL → 解析 → 路由 → 执行 → 归并 → 返回            │
└─────────────────────────────────────────────────────────────┘
```

---

## 7. 面试常见问题

### Q1：MyCAT2 为什么使用 Vertx？

**答**：Vertx 是一个高性能的非阻塞异步框架，能够支撑高并发的网络请求。使用 Vertx 可以避免传统的 BIO 模型的线程阻塞问题，提高 MyCAT2 的并发处理能力。

### Q2：MyCAT2 如何实现高可用？

**答**：通过配置集群（Cluster）和心跳检测（Heartbeat），当主节点故障时自动切换到从节点，保证服务的高可用性。

---

⬆️ [[索引]]
