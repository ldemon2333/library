emqx->dmp-mqtt->dmp-kernel->rocketmq->dmp-data->clickhouse
![[架构图 1.png]]
物联网平台的数据链路 **EMQX → DMP-MQTT → DMP-Kernel → RocketMQ → DMP-Data → ClickHouse** 是一个典型的物联网数据处理流程，涵盖设备连接、消息处理、业务逻辑、消息队列缓冲、数据持久化与分析等环节。以下是对各组件功能及链路优化的详细解析：

---

### **1. 数据链路各组件功能**
#### **（1）EMQX（MQTT Broker）**
- **角色**：物联网设备连接的入口，负责设备认证、消息路由和实时通信。
- **关键功能**：
  - 支持海量设备通过MQTT协议接入（TCP/TLS加密）。
  - 基于Topic的消息发布/订阅模型，实现设备与后端系统的解耦。
  - QoS等级保证消息可靠性（如QoS 1/2确保消息必达）。
  - 支持通过插件或规则引擎将数据转发至下游（如HTTP Webhook、Kafka、RocketMQ等）。

#### **（2）DMP-MQTT（数据管理平台接入层）**
- **角色**：协议转换与数据初步处理。
- **关键功能**：
  - **协议适配**：可能将MQTT消息转换为内部协议（如HTTP、gRPC）或标准化数据格式（JSON/Protobuf）。
  - **设备鉴权**：二次校验设备身份（如通过Token或证书），防止非法设备接入。
  - **数据过滤**：丢弃无效或重复数据（如设备心跳包）。
  - **流量控制**：限流或削峰填谷，保护下游系统。

#### **（3）DMP-Kernel（业务处理核心）**
- **角色**：执行核心业务逻辑与数据加工。
- **关键功能**：
  - **规则引擎**：触发告警（如温度超标）、设备联动（如关闭阀门）或调用外部API。
  - **数据标准化**：统一不同设备的数据格式（如单位转换、字段映射）。
  - **数据增强**：补充设备元数据（如从数据库查询设备位置信息）。
  - **缓存处理**：临时存储高频访问数据，减少数据库压力。

#### **（4）RocketMQ（消息队列）**
- **角色**：异步解耦与流量缓冲。
- **关键功能**：
  - **削峰填谷**：应对突发流量，避免ClickHouse写入过载。
  - **顺序性与可靠性**：保证消息按分区（Sharding Key）有序消费，确保至少一次投递。
  - **持久化存储**：消息落盘，避免系统崩溃导致数据丢失。

#### **（5）DMP-Data（数据处理与存储层）**
- **角色**：数据清洗与持久化。
- **关键功能**：
  - **数据清洗**：去噪（过滤异常值）、补全缺失字段、格式校验。
  - **批量聚合**：按时间窗口（如每分钟）聚合指标（如平均值、最大值）。
  - **数据分发**：将处理后的数据写入ClickHouse，同时可能同步到其他存储（如MySQL归档）。

#### **（6）ClickHouse（分析型数据库）**
- **角色**：海量数据存储与实时分析。
- **关键功能**：
  - **高效写入**：支持高吞吐批量插入（适合时序数据）。
  - **列式存储**：压缩率高，适合聚合查询（如统计某时间段设备总数）。
  - **分布式查询**：通过集群分片提升查询性能。
  - **TTL管理**：自动过期旧数据，节省存储成本。

---

### **2. 数据流向示例**
```plaintext
设备 → EMQX（MQTT Broker） 
       → DMP-MQTT（协议转换/过滤） 
          → DMP-Kernel（业务规则处理） 
             → RocketMQ（异步缓冲） 
                → DMP-Data（清洗/聚合） 
                   → ClickHouse（存储与分析）
```

---

### **3. 优化点与潜在问题**
#### **（1）性能优化**
- **EMQX集群化**：部署EMQX集群并启用共享订阅（Shared Subscription），避免单点瓶颈。
- **RocketMQ分区策略**：按设备ID或时间分区，提升并行消费能力。
- **ClickHouse写入批处理**：合并小批次写入（如每10秒批量插入），减少IO压力。
- **缓存利用**：在DMP-Kernel层使用Redis缓存设备元数据，降低数据库查询延迟。

#### **（2）可靠性保障**
- **消息确认机制**：确保EMQX到DMP-MQTT的消息投递（如MQTT QoS 1 + ACK确认）。
- **RocketMQ事务消息**：保证业务处理（DMP-Kernel）与消息发送的原子性。
- **ClickHouse副本机制**：使用ReplicatedMergeTree引擎防止数据丢失。

#### **（3）监控与运维**
- **链路追踪**：集成SkyWalking或Jaeger，跟踪消息全链路延迟。
- **指标监控**：
  - EMQX：连接数、消息吞吐、Topic堆积。
  - RocketMQ：消费延迟、积压消息数。
  - ClickHouse：Merge操作频率、ZooKeeper健康状态。
- **告警机制**：对关键指标（如RocketMQ积压超1小时）配置实时告警。

#### **（4）潜在问题**
- **数据热点**：设备数据分布不均导致RocketMQ分区负载倾斜，需合理设计Sharding Key。
- **时序数据冲突**：ClickHouse写入时可能出现重复时间戳，需通过`ReplacingMergeTree`去重。
- **资源竞争**：DMP-Kernel复杂计算可能阻塞消息消费，需异步化处理或拆分微服务。

---

### **4. 架构改进建议**
- **边缘计算**：在设备侧或近场网关（如EMQX Edge）预处理数据，减少云端传输量。
- **流处理引擎**：在DMP-Kernel层引入Flink或Kafka Streams，实现复杂事件处理（CEP）。
- **多级存储**：将冷数据从ClickHouse迁移至HDFS/S3，降低成本。

---

该架构已覆盖物联网核心场景，但需根据业务规模持续优化组件配置（如RocketMQ集群扩容、ClickHouse索引优化），并强化监控与灾备能力（如异地多活部署）。


你可以通过 **Prometheus + Grafana + 微服务日志** 结合 **分布式链路追踪** 来定位具体的瓶颈。你的架构是：

```
IoT 设备 -> EMQX -> dmp-mqtt -> dmp-kernel -> RocketMQ -> dmp-data -> ClickHouse
```

要找出瓶颈，主要分 **3 个维度**：

1. **基础监控**（CPU、内存、磁盘、网络）
2. **中间件性能监控**（EMQX、RocketMQ、ClickHouse）
3. **分布式链路追踪**（每个请求经过的微服务耗时）

---

# **📌 1. 基础监控**

**使用 Prometheus + Node Exporter 监控服务器资源**

- 监控 **CPU、内存、磁盘、网络**
- **重点关注：**
    - CPU 超高：可能是 ClickHouse 查询慢或 RocketMQ 消息堆积
    - 内存溢出：可能是 Kafka / RocketMQ / ClickHouse 配置不合理
    - 磁盘 IO 过高：ClickHouse 可能是瓶颈
    - 网络带宽过高：EMQX 或 RocketMQ 可能是瓶颈

**PromQL 查询示例**

- **CPU 使用率**
    
    ```promql
    100 - (avg by (instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)
    ```
    
- **磁盘 IO**
    
    ```promql
    rate(node_disk_read_bytes_total[5m]) + rate(node_disk_written_bytes_total[5m])
    ```
    
- **网络带宽**
    
    ```promql
    rate(node_network_receive_bytes_total[5m]) + rate(node_network_transmit_bytes_total[5m])
    ```
    

> **如果 CPU、磁盘、网络异常高，可能是系统瓶颈。否则，检查中间件。**

---

# **📌 2. 中间件性能监控**

### **🔹 监控 EMQX**

EMQX 提供 Prometheus 监控数据（默认端口 `18083`）。

- 访问 `http://EMQX_IP:18083/api/v4/metrics`
- 关注：
    - **消息吞吐量**
        
        ```promql
        rate(emqx_message_sent_total[5m])
        ```
        
    - **连接数**
        
        ```promql
        emqx_connections
        ```
        
    - **消息积压**
        
        ```promql
        emqx_retained_count
        ```
        
    - **TCP 连接异常**
        
        ```promql
        rate(emqx_tcp_closed[5m])
        ```
        

**👉 瓶颈表现：**

- **消息积压增加但吞吐量不变** → 可能是 **EMQX 消费者（dmp-mqtt）处理不过来**
- **连接数增长但 CPU 过载** → 可能是 **MQTT 连接数过多，超出负载**

---

### **🔹 监控 RocketMQ**

RocketMQ 提供 **Prometheus Exporter**（需部署 `rocketmq-exporter`）。

- 关注：
    - **队列长度**
        
        ```promql
        rate(rocketmq_broker_put_tps[5m])
        ```
        
    - **消息积压**
        
        ```promql
        rocketmq_broker_consume_queue_size
        ```
        
    - **消费耗时**
        
        ```promql
        rate(rocketmq_broker_get_latency[5m])
        ```
        

**👉 瓶颈表现：**

- **消息积压但消费 TPS 低** → 可能是 **消费者（dmp-kernel）处理不过来**
- **消息消费耗时高** → 可能是 **RocketMQ 消息存储瓶颈**

---

### **🔹 监控 ClickHouse**

ClickHouse 提供 `system.metrics` 监控：

- 关注：
    - **查询耗时**
        
        ```sql
        SELECT event_time, query_duration_ms FROM system.query_log ORDER BY event_time DESC LIMIT 10;
        ```
        
    - **磁盘 IO**
        
        ```sql
        SELECT metric, value FROM system.metrics WHERE metric LIKE '%Disk%';
        ```
        
    - **缓存命中率**
        
        ```sql
        SELECT metric, value FROM system.events WHERE metric LIKE 'Cache%';
        ```
        

**👉 瓶颈表现：**

- **查询时间变长但 CPU 低** → 可能是 **索引未优化**
- **磁盘 IO 过高** → 可能是 **查询量过大**
- **Cache 命中率低** → 可能是 **配置优化不足**

---

# **📌 3. 分布式链路追踪**

如果 **CPU/磁盘/网络 都正常，中间件也正常**，但系统仍然有瓶颈，可能是**业务代码性能问题**。可以用 **分布式链路追踪（Jaeger / SkyWalking）** 监控微服务调用链：

### **🔹 方案：Jaeger**

4. **在微服务（dmp-mqtt、dmp-kernel）添加 OpenTelemetry 代码**
5. **Jaeger 记录每个请求经过的微服务及耗时**
6. **如果某个微服务耗时异常高，即可确定瓶颈**

🚀 **示例：**

```python
from opentelemetry import trace

tracer = trace.get_tracer(__name__)

def process_message():
    with tracer.start_as_current_span("dmp-mqtt-process"):
        # 处理 MQTT 消息
        pass
```

- 这样，你可以在 **Jaeger UI** 看到：
    
    ```
    IoT -> EMQX [10ms] -> dmp-mqtt [50ms] -> dmp-kernel [5ms] -> RocketMQ [3ms] -> dmp-data [20ms] -> ClickHouse [100ms]
    ```
    
- **如果 ClickHouse 耗时 100ms，而 dmp-kernel 只有 5ms**，那瓶颈就是 ClickHouse！

---

# **📌 总结**

### **🔥 1. 服务器资源监控**

- **CPU/磁盘/网络瓶颈** → 可能是服务器负载问题
- 用 **Prometheus + Node Exporter** 监控 **CPU、磁盘 IO、网络**

### **🔥 2. 中间件监控**

- **EMQX：MQTT 连接数、吞吐量**
- **RocketMQ：消息积压、队列长度**
- **ClickHouse：查询耗时、磁盘 IO**
- **PromQL 查询发现是否有积压、队列拥堵、查询慢**

### **🔥 3. 分布式链路追踪**

- **用 Jaeger / SkyWalking 监控每个微服务耗时**
- **发现是哪一环最慢（EMQX、MQTT 处理、RocketMQ、ClickHouse）**

---

# **📌 实战步骤**

✅ **Step 1：用 Prometheus + Grafana 监控 CPU / 内存 / 磁盘 IO**  
✅ **Step 2：检查 EMQX / RocketMQ / ClickHouse 的吞吐量和积压情况**  
✅ **Step 3：用 Jaeger 分析业务请求链路，找到最慢的环节**

🚀 这样，你就能快速找到 IoT 平台的瓶颈并优化！