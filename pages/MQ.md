public:: true

- ## RabbitMQ 跟 kafka 差異
	- **一句話回答：**
	  "RabbitMQ 是傳統 message broker，適合複雜路由和保證送達；Kafka 是 distributed streaming platform，適合高吞吐量和 event sourcing。"
- ## 詳細對比
	- #### A. 設計理念
	  collapsed:: true
		- | 特性 | RabbitMQ | Kafka |
		  |------|----------|-------|
		  | **定位** | Message Broker | Distributed Log |
		  | **使用場景** | Task queue, RPC | Event streaming, Log aggregation |
		  | **訊息模型** | Push (broker push to consumer) | Pull (consumer pull from broker) |
		  | **訊息保留** | 消費後刪除 | 保留一段時間（可設定） |
		  | **順序保證** | Queue level | Partition level |
	- #### B. 架構差異
	  collapsed:: true
		- **RabbitMQ:**
		  collapsed:: true
		  ```
		  Producer → Exchange → Queue(s) → Consumer(s)
		             ↓
		          (routing)
		  ```
			- **Exchange types**: Direct, Topic, Fanout, Headers
			- **複雜路由**：可以用 routing key 做靈活的訊息分配
			- **Multiple queues per consumer**
		- **Kafka:**
		  collapsed:: true
		  ```
		  Producer → Topic (multiple partitions) → Consumer Group
		  ```
			- **Simple routing**：只有 topic 和 partition
			- **Partition = ordered log**
			- **Replication** 內建
	- #### C. 使用場景對比
	  collapsed:: true
		- **RabbitMQ 適合：**
		  collapsed:: true
			- collapsed:: true
			  1. **Task Queue**
				- Worker pattern
				- 保證 exactly-once processing（搭配 acknowledgment）
			- collapsed:: true
			  2. **複雜 Routing**
				- 需要根據 routing key 送到不同 queue
				- Topic exchange 做靈活的訂閱
			- collapsed:: true
			  3. **RPC Pattern**
				- Request-reply 模式
				- 需要回覆的場景
			- collapsed:: true
			  4. **低延遲優先**
				- Sub-millisecond latency
		- **Kafka 適合：**
		  collapsed:: true
			- collapsed:: true
			  1. **High Throughput**
				- 每秒處理百萬級訊息
			- collapsed:: true
			  2. **Event Sourcing**
				- 需要保留所有 events
				- Event replay
			- collapsed:: true
			  3. **Log Aggregation**
				- 收集大量 logs
				- 數據 pipeline
			- collapsed:: true
			  4. **Stream Processing**
				- Real-time analytics
				- Data transformation
	- #### D. 核心技術差異
	  collapsed:: true
		- **1. Message 保留**
		  collapsed:: true
		  ```
		  RabbitMQ:
		  Producer → Queue → Consumer → [Message deleted]
		  
		  Kafka:
		  Producer → Partition → Consumers → [Message still there]
		                                    (保留 7 days)
		  ```
		  
		  **影響：**
			- RabbitMQ: 消費後訊息就沒了，不能 replay
			- Kafka: 可以 replay，可以讓新的 consumer 從頭讀
		- **2. Consumer 模型**
		  ```
		  RabbitMQ - Push:
		  Broker 主動推給 Consumer
		  → Consumer 需要處理 back pressure
		  
		  Kafka - Pull:
		  Consumer 主動拉取
		  → Consumer 控制速度
		  → 可以 batch 拉取
		  ```
		- **3. Ordering**
		  ```
		  RabbitMQ:
		  在 single queue 內保證順序
		  Multiple consumers 可能打亂
		  - Kafka:
		  在 single partition 內保證順序
		  Partition key 決定哪個 partition
		  ```
		- **4. Scalability**
		  ```
		  RabbitMQ:
		  Vertical scaling (單機效能)
		  Clustering 有限
		  - Kafka:
		  Horizontal scaling (加 broker)
		  Partition 分散負載
		  ```
- ## 面試回答模板
	- ### 什麼時候用 RabbitMQ，什麼時候用 Kafka
	  collapsed:: true
		- 用 RabbitMQ：
		  collapsed:: true
			- Task queue（worker pattern）
			- 需要複雜 routing
			- RPC pattern
			- 低延遲更重要
		- 用 Kafka：
		  collapsed:: true
			- High throughput（百萬 msg/sec）
			- 需要 replay events
			- Event sourcing
			- Log aggregation
			- Data pipeline
	- ### Kafka 的 Consumer Group 是什麼？
	  collapsed:: true
		- Consumer Group 讓多個 consumer 共同消費一個 topic。
		- Key 概念：
		  collapsed:: true
			- 同一個 group 內，每個 partition 只能被一個 consumer 消費
			- 不同 group 可以獨立消費同一個 topic
			- 用來做 horizontal scaling
			  
			  比如有 Topic 有 4 個 partitions，Consumer Group 有 2 個 consumers，那麼每個 consumer 會負責 2 個 partitions。"
	- ### RabbitMQ 的 Exchange 類型？
	  collapsed:: true
		- 1. Direct: 完全匹配 routing key
		  2. Topic: Pattern 匹配 `(用 * 和 #)`
		  3. Fanout: 廣播給所有綁定的 queue
		  4. Headers: 根據 message headers 路由
	- ### Kafka 的 replication 怎麼運作？
	  collapsed:: true
		- Leader/Follower
		- ISR (In-Sync Replicas)
		- 保證 durability
	- ### RabbitMQ 怎麼保證訊息不丟？
	  collapsed:: true
		- Publisher confirms
		- Consumer acknowledgments
		- Durable queues
		- Persistent messages