# 04 性能优化与 Mapping 设计

> 双链：[[38_ElasticSearchKnowledge/03_分布式架构与写查流程]] | [[38_ElasticSearchKnowledge/05_运维与监控]]

---

## 1. 写入性能优化

### 1.1 批量写入（Bulk API）

**单条写入 vs 批量写入：单次 Bulk 请求省去大量网络往返开销。**

```java
// 推荐：批量写入
BulkRequest.Builder bulk = new BulkRequest.Builder();
for (Product p : products) {
    bulk.operations(op -> op.index(i -> i.index("products").id(p.getId()).document(p)));
}
esClient.bulk(bulk.build());
```

**Bulk 最佳实践：**
- 每个 Bulk 请求大小：**5MB ~ 15MB**（不是按条数，按总字节数）
- 每批次条数：**500 ~ 5000 条**（视文档大小而定）
- 压测不同 bulk size，找到最优值
- 使用多线程并发写入（充分利用多分片）

### 1.2 索引刷新间隔

```json
// 批量导入时：关闭自动 refresh（提升 3~5x 写入速度）
PUT /products/_settings
{"index.refresh_interval": "-1"}

// 导入完成后：恢复正常
PUT /products/_settings
{"index.refresh_interval": "1s"}
```

### 1.3 副本策略

```json
// 初始全量导入时：临时去掉副本（减少 50% 写入开销）
PUT /products/_settings
{"index.number_of_replicas": 0}

// 导入完成后恢复副本
PUT /products/_settings
{"index.number_of_replicas": 1}
```

### 1.4 Translog 异步模式

```json
// 允许丢失最多 sync_interval 时间内的数据（换取高吞吐）
PUT /products/_settings
{
  "index.translog.durability": "async",
  "index.translog.sync_interval": "5s"
}
```

**注意**：异步模式在节点宕机时可能丢失 5 秒内的数据，根据业务容忍度决定是否开启。

### 1.5 其他写入优化

```json
// 禁用 _source 存储（日志场景，只用于统计，不需要返回原文档）
"_source": {"enabled": false}

// 禁用字段索引（某些字段只存储不搜索）
"big_field": {"type": "text", "index": false}

// 使用压缩编码（磁盘占用减少 20-40%，略微增加 CPU）
PUT /products/_settings
{"index.codec": "best_compression"}
```

---

## 2. 查询性能优化

### 2.1 优先使用 Filter

```json
// 差：match 在 query context，每次计算评分，不缓存
{"query": {"match": {"status": "active"}}}

// 好：term 在 filter context，可缓存，快 10x+
{"query": {"bool": {"filter": [{"term": {"status": "active"}}]}}}
```

**Filter 缓存（Node Query Cache）**：LRU 缓存，默认大小为堆内存的 10%，缓存命中后完全跳过倒排索引查找。

### 2.2 避免深分页

```json
// 禁止深分页（前端不应该跳到第 10000 页）
// 用 search_after 游标翻页
GET /products/_search
{
  "sort": [{"_id": "asc"}],
  "search_after": ["last_id_from_previous_page"],
  "size": 20
}
```

### 2.3 控制返回字段

```json
GET /products/_search
{
  "_source": ["title", "price", "brand"],   // 只返回需要的字段
  "query": {"match": {"title": "手机"}}
}

// 完全禁用 _source（当 _source: disabled 时无法高亮、Reindex）
"_source": false
```

### 2.4 路由优化

```json
// 查询时指定路由，只扫描特定分片
GET /orders/_search?routing=user_123
{
  "query": {"term": {"user_id": "user_123"}}
}
```

### 2.5 避免高基数聚合

```json
// 差：对高基数字段（几百万不同值）聚合，内存开销极大
"aggs": {"by_email": {"terms": {"field": "email"}}}

// 好：限制聚合返回数量
"aggs": {"by_category": {"terms": {"field": "category", "size": 20}}}

// 更好：使用 cardinality 聚合只获取近似去重数
"aggs": {"unique_users": {"cardinality": {"field": "user_id"}}}
```

### 2.6 利用 Profile API 分析慢查询

```json
GET /products/_search
{
  "profile": true,
  "query": {
    "bool": {
      "must": [{"match": {"title": "手机"}}],
      "filter": [{"term": {"category": "数码"}}]
    }
  }
}
```

**Profile 结果解读：**
- `time_in_nanos`：各查询组件耗时
- `next_doc`：迭代匹配文档的耗时
- `score`：评分计算耗时
- `match`：实际匹配次数

---

## 3. JVM 调优

### 3.1 堆内存设置

```bash
# /etc/elasticsearch/jvm.options
# 规则：
# 1. 设为物理内存的 50%（另外 50% 留给 OS 文件系统缓存）
# 2. 不超过 32GB（超过 32GB 关闭 Compressed Ordinary Object Pointers，内存效率反而下降）
# 3. Xms == Xmx（固定堆大小，避免 JVM 动态扩缩堆时的 GC 停顿）

-Xms16g
-Xmx16g
```

**32GB 限制详解：**
- JVM 堆 < 32GB：使用 Compressed OOPs，对象引用 4 字节
- JVM 堆 > 32GB：关闭 Compressed OOPs，对象引用 8 字节，内存效率降低约 50%
- 因此 30GB 堆 可能比 32GB 堆能存更多数据

### 3.2 GC 策略

```bash
# ES 8.x 默认使用 G1GC（推荐）
-XX:+UseG1GC
-XX:G1HeapRegionSize=4m
-XX:InitiatingHeapOccupancyPercent=30   # 堆占用 30% 时开始并发标记

# ZGC（ES 7.x+ 可选，适合超大堆，低延迟）
-XX:+UseZGC
```

**GC 问题排查：**

```bash
# 查看 GC 日志
cat /var/log/elasticsearch/gc.log | grep -E "GC|pause"

# JVM 堆使用情况
GET /_nodes/stats/jvm?filter_path=nodes.*.jvm.mem,nodes.*.jvm.gc
```

### 3.3 Circuit Breaker（熔断器）

**防止大查询/聚合导致 OOM：**

```json
GET /_nodes/stats/breaker  // 查看熔断器状态

PUT /_cluster/settings
{
  "persistent": {
    "indices.breaker.total.limit": "95%",         // 总内存限制
    "indices.breaker.fielddata.limit": "40%",     // Fielddata 限制
    "indices.breaker.request.limit": "60%",       // 单次请求限制
    "network.breaker.inflight_requests.limit": "100%" // 传输中请求限制
  }
}
```

---

## 4. Linux 系统优化

```bash
# 1. 禁用 swap（ES 对 swap 非常敏感）
swapoff -a
# 永久禁用：注释 /etc/fstab 中的 swap 行

# 2. 增大虚拟内存映射数（Lucene 使用 mmap）
sysctl -w vm.max_map_count=262144
# 永久：echo "vm.max_map_count=262144" >> /etc/sysctl.conf

# 3. 增大文件描述符限制
ulimit -n 65535
# /etc/security/limits.conf:
# elasticsearch soft nofile 65535
# elasticsearch hard nofile 65535

# 4. 增大线程数
ulimit -u 4096
# elasticsearch soft nproc 4096

# 5. 锁定内存（防止 ES 内存被 swap 到磁盘）
# elasticsearch.yml: bootstrap.memory_lock: true
# /etc/security/limits.conf:
# elasticsearch soft memlock unlimited
# elasticsearch hard memlock unlimited

# 6. 磁盘调度算法（SSD 用 noop/deadline，机械硬盘用 deadline）
echo noop > /sys/block/sda/queue/scheduler
```

---

## 5. Mapping 设计最佳实践

### 5.1 禁止 Dynamic Mapping

```json
PUT /my-index
{
  "mappings": {
    "dynamic": "strict",  // 新字段直接报错
    "properties": {}
  }
}
```

**为什么禁止 Dynamic Mapping？**
- 字符串字段默认同时创建 `text` + `keyword` 两种类型，浪费存储
- 数字可能被识别为 `long` 而实际只需 `integer`
- 一旦 Mapping 确定，不能修改（只能新增字段）

### 5.2 字段类型选择指南

```
问题1：该字段需要全文搜索吗？
  是 → text（+analyzer）
  否 → keyword（精确匹配/排序/聚合）

问题2：既需要搜索又需要聚合？
  → text + keyword 子字段（multi-fields）

问题3：数值字段用于范围查询还是精确匹配？
  精确匹配 → keyword（"订单状态码" 等）
  范围查询 → integer/long/double

问题4：时间字段存什么格式？
  → date 类型，format 明确指定格式

问题5：JSON 对象如何存？
  对象不需要独立查询 → object（默认）
  对象数组需要保持独立性 → nested（慎用，写入慢）
```

### 5.3 避免 Mapping 爆炸（Mapping Explosion）

**问题**：动态 Mapping 下，用户随意写入新字段，导致 Mapping 有数千个字段，集群状态（cluster state）膨胀，Master 节点 OOM。

**解决：**

```json
// 限制索引字段总数
PUT /my-index/_settings
{"index.mapping.total_fields.limit": 200}

// 使用 flattened 类型处理动态 key
"metadata": {"type": "flattened"}
// 整个 metadata 对象视为一个字段，不展开子字段
```

### 5.4 Text 字段优化

```json
// 日志场景：message 只搜索，不需要高亮/排序 → 用 match_only_text
"message": {"type": "match_only_text"}

// 搜索场景：写入用细粒度分词，搜索用粗粒度（减少误召回）
"title": {
  "type": "text",
  "analyzer": "ik_max_word",
  "search_analyzer": "ik_smart"
}

// 不需要搜索的大文本字段：只存储，不建索引
"raw_content": {"type": "text", "index": false}

// 需要聚合或排序：添加 keyword 子字段
"title": {
  "type": "text",
  "fields": {
    "keyword": {"type": "keyword", "ignore_above": 256}
  }
}
```

### 5.5 数值字段优化

```json
// 金额用 scaled_float（避免浮点精度问题）
"price": {"type": "scaled_float", "scaling_factor": 100}
// 存储时：5999 → 599900（整数存储），读取时自动/100

// 不参与范围查询的数字用 keyword（如：手机号、邮编）
"phone": {"type": "keyword"}

// 整数类型尽量精确（节省存储）
"age": {"type": "byte"}          // 年龄：-128~127 够用
"quantity": {"type": "short"}    // 库存：-32768~32767
```

---

## 6. 慢查询日志与性能分析

### 6.1 慢查询日志配置

```json
PUT /my-index/_settings
{
  "index.search.slowlog.threshold.query.warn": "5s",
  "index.search.slowlog.threshold.query.info": "2s",
  "index.search.slowlog.threshold.query.debug": "500ms",
  "index.search.slowlog.threshold.fetch.warn": "1s",
  "index.search.slowlog.threshold.fetch.info": "500ms",
  "index.search.slowlog.level": "info",
  "index.indexing.slowlog.threshold.index.warn": "10s",
  "index.indexing.slowlog.threshold.index.info": "5s"
}
```

**慢日志路径：**
- 查询慢日志：`/var/log/elasticsearch/<cluster>_index_search_slowlog.json`
- 写入慢日志：`/var/log/elasticsearch/<cluster>_index_indexing_slowlog.json`

### 6.2 性能诊断命令速查

```bash
# 1. 热线程（找 CPU 占用高的线程）
GET /_nodes/hot_threads?type=cpu&interval=500ms

# 2. 线程池积压（找 rejected 请求）
GET /_cat/thread_pool?v&h=name,active,queue,rejected,completed&s=rejected:desc

# 3. 索引统计（查询/写入耗时）
GET /my-index/_stats?filter_path=indices.*.total.search,indices.*.total.indexing

# 4. 分片统计
GET /_cat/shards?v&h=index,shard,prirep,docs,store,segments.count&s=segments.count:desc

# 5. 查看 Fielddata 使用
GET /_cat/fielddata?v&s=size:desc

# 6. 节点资源
GET /_cat/nodes?v&h=name,heap.percent,ram.percent,cpu,load_1m,disk.used_percent
```
