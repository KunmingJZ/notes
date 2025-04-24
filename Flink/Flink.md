Apache Flink 是一个用于**大规模数据流处理**的开源框架，适合做 **流处理（stream processing）** 和 **批处理（batch processing）**。它以高吞吐、低延迟、精确一次（exactly-once）语义等特性而著称，非常适合处理实时大数据应用，比如实时日志分析、实时推荐系统、实时告警系统等。

下面是 Flink 的一些基础知识，帮助你快速了解这个框架：

---

### 📌 一、Flink 的核心概念

|概念|简要说明|
|---|---|
|**流（Stream）**|无界或有界的数据序列，例如实时日志、传感器数据等|
|**数据流（DataStream）**|Flink 中用于表示流数据的抽象，支持各种转换操作|
|**数据集（DataSet）**|（Flink 的旧 API）用于批处理的抽象，已逐渐被 Table API 取代|
|**作业（Job）**|Flink 提交执行的一个程序，包括源、转换和接收器|
|**算子（Operator）**|转换数据的步骤，比如 map、filter、keyBy、window 等|
|**状态（State）**|Flink 中保存跨事件信息的机制（比如流量累计）|
|**检查点（Checkpoint）**|保证容错性，保存作业的中间状态，支持恢复|
|**时间语义（Time Semantics）**|支持三种：处理时间（Processing Time）、事件时间（Event Time）、摄取时间（Ingestion Time）|

---

### 🧩 二、Flink 的核心组件

|组件名|功能|
|---|---|
|**JobManager**|协调作业的调度与资源分配，类似 Master|
|**TaskManager**|执行实际任务（子任务），类似 Worker|
|**Client**|提交作业、与 JobManager 交互|
|**Checkpoint Coordinator**|控制检查点的触发和完成，确保数据一致性|

---

### 🔧 三、常用 API

Flink 提供了三种主要的 API：

1. **DataStream API**：用于构建流式应用，是最常用的 API。
    
2. **Table API / SQL**：适合熟悉 SQL 的开发者，可用于流和批处理。
    
3. **ProcessFunction**：底层 API，适合需要细粒度控制的场景（如自定义状态、时间等）。
    

---

### 📦 四、常用操作示例（DataStream API）

java

![[Pasted image 20250424181406.png]]



---

### 🧠 五、Flink 的时间与窗口（Window）

- **时间类型**：
    
    - 事件时间（Event Time）：使用事件中携带的时间戳。
        
    - 处理时间（Processing Time）：使用系统当前时间。
        
- **窗口类型**：
    
    - 滚动窗口（Tumbling Window）
        
    - 滑动窗口（Sliding Window）
        
    - 会话窗口（Session Window）
        

---

### 🛡️ 六、Flink 的容错机制

- **状态后端（State Backend）**：内存、RocksDB 等
    
- **检查点（Checkpoint）**：周期性保存任务状态
    
- **重启策略（Restart Strategy）**：失败后的恢复方式设置
    
- **Exactly-once 语义**：保证数据只处理一次（结合 Kafka、Checkpoint 实现）
## 官网
     https://flink.apache.org/