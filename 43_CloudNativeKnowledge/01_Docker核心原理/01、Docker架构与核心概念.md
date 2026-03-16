# Docker架构与核心概念

## 一、Docker概述
### 1.1 什么是Docker
Docker是一个开源的容器化平台，用于开发、交付和运行应用程序。它将应用程序及其依赖项打包到一个轻量级、可移植的容器中。

### 1.2 Docker的优势
- **一致性环境**：开发、测试、生产环境一致
- **隔离性**：应用程序相互隔离
- **轻量级**：共享主机内核，资源占用少
- **快速部署**：秒级启动和停止
- **易于扩展**：水平扩展简单

## 二、Docker架构
### 2.1 整体架构
```
┌─────────────────────────────────┐
│           Docker Client           │
└───────────────────┬─────────────┘
                    │ REST API
┌───────────────────▼─────────────┐
│          Docker Daemon          │
└───────────────────┬─────────────┘
                    │
┌───────────────────▼─────────────┐
│        Containerd (运行时)       │
└───────────────────┬─────────────┘
                    │
┌───────────────────▼─────────────┐
│        runc (容器运行时)         │
└───────────────────┬─────────────┘
                    │
┌───────────────────▼─────────────┐
│        Linux Kernel            │
└─────────────────────────────────┘
```

### 2.2 核心组件
1. **Docker Client**：命令行工具，与Docker Daemon通信
2. **Docker Daemon**：后台服务，管理容器、镜像、网络等
3. **Containerd**：容器运行时管理
4. **runc**：底层容器运行时，遵循OCI标准
5. **Docker Registry**：镜像仓库（如Docker Hub）

## 三、核心概念
### 3.1 镜像（Image）
- 只读的模板，包含运行应用程序所需的一切
- 分层存储结构，每一层都是只读的
- 通过Dockerfile构建

### 3.2 容器（Container）
- 镜像的运行实例
- 可读写的容器层（Copy-on-Write）
- 包含运行时的应用程序进程

### 3.3 仓库（Registry）
- 存储和分发镜像的地方
- 公共仓库：Docker Hub
- 私有仓库：Harbor、Nexus等

## 四、Docker核心技术
### 4.1 命名空间（Namespaces）
Docker使用Linux命名空间实现隔离：
- **PID命名空间**：进程隔离
- **NET命名空间**：网络隔离
- **IPC命名空间**：进程间通信隔离
- **MNT命名空间**：文件系统挂载点隔离
- **UTS命名空间**：主机名和域名隔离
- **User命名空间**：用户和用户组隔离

### 4.2 控制组（Cgroups）
用于资源限制和统计：
- **CPU限制**：CPU份额、CPU集
- **内存限制**：内存使用量、交换内存
- **I/O限制**：磁盘I/O带宽
- **设备访问控制**

### 4.3 联合文件系统（UnionFS）
实现镜像的分层存储：
- **底层只读层**：基础镜像层
- **可写层**：容器运行时修改
- **Copy-on-Write**：写时复制机制

支持的存储驱动：
- overlay2（推荐）
- aufs
- devicemapper
- btrfs
- zfs

## 五、Docker工作流程
### 5.1 镜像构建
```bash
# 编写Dockerfile
FROM ubuntu:20.04
RUN apt-get update && apt-get install -y nginx
COPY index.html /var/www/html/
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]

# 构建镜像
docker build -t my-nginx .

# 运行容器
docker run -d -p 8080:80 my-nginx
```

### 5.2 容器生命周期
```bash
# 启动容器
docker run [options] image [command]

# 查看运行中的容器
docker ps

# 停止容器
docker stop container_id

# 删除容器
docker rm container_id

# 进入容器
docker exec -it container_id /bin/bash
```

## 六、网络模式
### 6.1 网络类型
1. **bridge**：默认网络模式，NAT网络
2. **host**：使用主机网络栈
3. **none**：无网络
4. **container**：共享其他容器的网络栈
5. **overlay**：多主机网络

### 6.2 网络配置
```bash
# 创建自定义网络
docker network create my-network

# 运行容器并使用自定义网络
docker run --network=my-network my-app
```

## 七、数据管理
### 7.1 数据卷（Volumes）
```bash
# 创建数据卷
docker volume create my-volume

# 使用数据卷
docker run -v my-volume:/data my-app

# 绑定挂载（Bind Mounts）
docker run -v /host/path:/container/path my-app
```

### 7.2 数据卷容器
```bash
# 创建数据卷容器
docker create -v /data --name data-container busybox

# 使用数据卷容器
docker run --volumes-from data-container my-app
```

## 八、安全最佳实践
1. 使用非root用户运行容器
2. 定期更新基础镜像
3. 扫描镜像中的漏洞
4. 限制容器资源使用
5. 使用只读文件系统
6. 避免在镜像中存储敏感信息

## 九、面试常见问题
1. Docker和虚拟机的区别？
2. Docker的架构是怎样的？
3. 什么是镜像的分层存储？
4. Docker使用哪些Linux技术实现隔离？
5. 如何优化Docker镜像大小？
6. Docker网络模式有哪些？

## 十、性能优化
1. 使用多阶段构建减少镜像大小
2. 合理使用.dockerignore文件
3. 利用构建缓存
4. 选择合适的基础镜像
5. 合并RUN指令减少层数