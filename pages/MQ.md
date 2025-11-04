public:: true

- ## RabbitMQ 跟 kafka 差異
	- **一句話回答：**
	  "RabbitMQ 是傳統 message broker，適合複雜路由和保證送達；Kafka 是 distributed streaming platform，適合高吞吐量和 event sourcing。"
		- **Event Sourcing** 是一種架構模式，核心概念是：**不儲存資料的最終狀態，而是儲存所有改變資料的事件（events）**
- ## 詳細對比
	- #### A. 設計理念
		- | 特性 | RabbitMQ | Kafka |
		  |------|----------|-------|
		  | **定位** | Message Broker | Distributed Log |
		  | **使用場景** | Task queue, RPC | Event streaming, Log aggregation |
		  | **訊息模型** | Push (broker push to consumer) | Pull (consumer pull from broker) |
		  | **訊息保留** | 消費後刪除 | 保留一段時間（可設定） |
		  | **順序保證** | Queue level | Partition level |
	- #### B. 架構差異
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
		- **RabbitMQ 適合：**
			- 1. **Task Queue**
				- Worker pattern
				- 保證 exactly-once processing（搭配 acknowledgment）
			- 2. **複雜 Routing**
				- 需要根據 routing key 送到不同 queue
				- Topic exchange 做靈活的訂閱
			- 3. **RPC Pattern**
				- Request-reply 模式
				- 需要回覆的場景
			- 4. **低延遲優先**
				- Sub-millisecond latency
		- **Kafka 適合：**
			- 1. **High Throughput**
				- 每秒處理百萬級訊息
			- 2. **Event Sourcing**
				- 需要保留所有 events
				- Event replay
			- 3. **Log Aggregation**
				- 收集大量 logs
				- 數據 pipeline
			- 4. **Stream Processing**
				- Real-time analytics
				- Data transformation
	- #### D. 核心技術差異
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
		  → Consumer 需要處理 back pressure (如果 Consumer 處理太慢 → 訊息堆積在 Consumer 端)
		  
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
		- 用 RabbitMQ：
			- Task queue（worker pattern）
			- 需要複雜 routing
			- RPC pattern
			- 低延遲更重要
		- 用 Kafka：
			- High throughput（百萬 msg/sec）
			- 需要 replay events
			- Event sourcing
			- Log aggregation
			- Data pipeline
	- ### Kafka 的 Consumer Group 是什麼？
		- Consumer Group 讓多個 consumer 共同消費一個 topic。
		- Key 概念：
			- 同一個 group 內，每個 partition 只能被一個 consumer 消費
			- 不同 group 可以獨立消費同一個 topic
			- 用來做 horizontal scaling
			  
			  比如有 Topic 有 4 個 partitions，Consumer Group 有 2 個 consumers，那麼每個 consumer 會負責 2 個 partitions。"
	- ### RabbitMQ 的 Exchange 類型？
		- 1. Direct: 完全匹配 routing key
		  2. Topic: Pattern 匹配 `(用 * 和 #)`
		  3. Fanout: 廣播給所有綁定的 queue
		  4. Headers: 根據 message headers 路由
	- ### Kafka 的 replication 怎麼運作？
		- Leader/Follower
		   ```
		  Topic: user-events (3 partitions, replication factor = 3)
		  
		  Partition 0:
		    Leader: Broker 1  ← 處理所有讀寫
		    Follower: Broker 2  ← 複製 Leader 的資料
		    Follower: Broker 3  ← 複製 Leader 的資料
		  
		  Partition 1:
		    Leader: Broker 2
		    Follower: Broker 1
		    Follower: Broker 3
		  
		  Partition 2:
		    Leader: Broker 3
		    Follower: Broker 1
		    Follower: Broker 2
		  ```
			- 只有 Leader 處理讀寫請求
			  ```
			  Producer → 只能寫到 Leader
			  Consumer → 只能從 Leader 讀（Kafka 2.4+ 可以從 Follower 讀）
			  
			  Follower 的工作：
			  - 從 Leader 拉取資料（像個 Consumer）
			  - 保持資料同步
			  - Leader 掛了，可以被選為新 Leader
			  ```
		- ISR (In-Sync Replicas)
			- **ISR = 跟得上 Leader 的 Followers**
			  ```
			  Partition 0:
			    Leader: Broker 1 (offset: 1000)
			    ISR: [Broker 1, Broker 2, Broker 3]  ← 都跟得上
			    
			    Broker 2 (offset: 1000)  ← 完全同步
			    Broker 3 (offset: 998)   ← 稍微落後，但在容忍範圍內
			  ```
			- ISR 的判斷標準
			  兩個條件**都要滿足**才算 In-Sync：
				- 1. **replica.lag.time.max.ms**（預設 10 秒）
				  ```
				   如果 Follower 超過 10 秒沒向 Leader 發送 fetch request
				   → 踢出 ISR
				  ```
				  
				  2. **Follower 的 offset 不能落後太多**
				  ```
				   Leader offset: 1000
				   Follower offset: 990  ← 如果這個差距在容忍範圍內，就算 In-Sync
				  ```
			- ISR 動態變化的例子
			  ```
			  初始狀態：
			  ISR: [Broker 1(L), Broker 2, Broker 3]
			  
			  Broker 3 網路變慢：
			  → 超過 10 秒沒 fetch
			  → ISR: [Broker 1(L), Broker 2]  ← Broker 3 被踢出
			  
			  Broker 3 網路恢復：
			  → 追上進度
			  → ISR: [Broker 1(L), Broker 2, Broker 3]  ← 重新加入
			  ```
		- 保證 durability
			- | acks 值 | 行為 | Durability | 效能 |
			  |---------|------|------------|------|
			  | **0** | 不等任何確認 | ❌ 最弱（可能丟） | ⚡ 最快 |
			  | **1** | 等 Leader 寫入 | ⚠️ 中等（Leader 掛可能丟） | 🔥 快 |
			  | **all/-1** | 等所有 ISR 確認 | ✅ 最強 | 🐢 較慢 |
			-
	- ### RabbitMQ 怎麼保證訊息不丟？
		- Publisher confirms
		- Consumer acknowledgments
		- Durable queues
		- Persistent messages
- # [[Kafka]]