# Docker 快速搭建 MySQL 主从

## 1. Docker 环境准备

### 拉取 MySQL 镜像

```bash
docker pull mysql
```

### 启动主节点容器

```bash
docker run --name mysql-master \
  -e MYSQL_ROOT_PASSWORD=root \
  -v ~/Desktop/data/master-mysql:/var/lib/mysql \
  -p 3306:3306 \
  -d mysql
```

### 启动从节点容器

```bash
docker run --name mysql-slave \
  -e MYSQL_ROOT_PASSWORD=root \
  -v ~/Desktop/data/slave-mysql:/var/lib/mysql \
  -p 3308:3306 \
  -d mysql
```

---

## 2. 主服务器配置

### 2.1 修改配置文件

```bash
# 复制配置文件到本地
docker cp mysql-master:/etc/mysql/my.cnf ./my.cnf

# 添加 server_id（如果没有配置的话）
# 在 [mysqld] 下添加：
server-id=1
log-bin=mysql-bin

# 复制回容器
docker cp ./my.cnf mysql-master:/etc/mysql/my.cnf

# 重启
docker restart mysql-master
```

### 2.2 创建复制用户

```bash
# 登录 MySQL
docker exec -it mysql-master mysql -u root -proot

# 执行 SQL
CREATE USER 'replication_user'@'%' IDENTIFIED WITH mysql_native_password BY 'root';
GRANT REPLICATION SLAVE ON *.* TO 'replication_user'@'%';
FLUSH PRIVILEGES;
```

### 2.3 查看主服务器状态

```sql
SHOW MASTER STATUS;
-- 记录 File 和 Position 的值，后续配置从服务器需要使用
```

---

## 3. 从服务器配置

### 3.1 修改配置文件

```bash
# 复制配置文件到本地
docker cp mysql-slave:/etc/mysql/my.cnf ./my.cnf

# 添加 server-id（确保与主服务器不同）
server-id=2

# 复制回容器
docker cp ./my.cnf mysql-slave:/etc/mysql/my.cnf

# 重启
docker restart mysql-slave
```

### 3.2 配置主从复制

```bash
# 登录 MySQL
docker exec -it mysql-slave mysql -u root -proot

# 配置复制
CHANGE REPLICATION SOURCE TO 
    SOURCE_HOST='172.17.0.2',  -- 主服务器 IP
    SOURCE_USER='replication_user',
    SOURCE_PASSWORD='root',
    SOURCE_LOG_FILE='binlog.000005',  -- 主服务器 SHOW MASTER STATUS 中的值
    SOURCE_LOG_POS=4566;              -- 主服务器 SHOW MASTER STATUS 中的值
```

### 3.3 启动复制并查看状态

```sql
-- 启动复制
START REPLICA;

-- 查看状态
SHOW REPLICA STATUS\G
```

---

## 4. 测试主从复制

### 4.1 在主库创建测试数据库

```sql
-- 登录主库
docker exec -it mysql-master mysql -u root -proot

-- 创建数据库
CREATE DATABASE mydatabase;

-- 创建表
CREATE TABLE `test` (
  `id` int(11) unsigned NOT NULL AUTO_INCREMENT COMMENT '主键',
  `remark` varchar(255) DEFAULT NULL COMMENT '说明',
  PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8 COMMENT='test';

-- 插入数据
INSERT INTO mydatabase.test VALUES (22, 'test');
```

### 4.2 在从库验证

```sql
-- 登录从库
docker exec -it mysql-slave mysql -u root -proot

-- 查询数据
SELECT * FROM mydatabase.test;
```

如果能看到主库插入的数据，说明主从复制配置成功。

---

## 5. 常见问题处理

### 问题1：复制失败

```sql
-- 如果需要重新配置
STOP REPLICA;
RESET REPLICA;

-- 重新配置
CHANGE REPLICATION SOURCE TO ...
START REPLICA;
```

### 问题2：查看从库延迟

```sql
SHOW REPLICA STATUS\G
-- 查看 Seconds_Behind_Master 字段
```

### 问题3：IO线程连接失败

检查：
1. 主服务器防火墙是否开放端口
2. 复制用户权限是否正确
3. 主服务器 IP 地址是否正确

---

## 6. 双主双从搭建（扩展）

### 架构说明

```
┌──────────────┐         ┌──────────────┐
│  Master1     │◄───────►│  Master2     │
│  (3307)      │         │  (3309)      │
└──────┬───────┘         └──────┬───────┘
       │                        │
       ▼                        ▼
┌──────────────┐         ┌──────────────┐
│  Slave1      │◄───────►│  Slave2      │
│  (3308)      │         │  (3310)      │
└──────────────┘         └──────────────┘
```

### 配置要点

1. 两个主服务器都开启 `log-bin` 和 `server-id`
2. 互相配置对方为自己的主服务器
3. 从服务器分别指向对应的 Master

---

⬆️ [[索引]]
