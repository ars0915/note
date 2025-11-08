## 說明
	- ```
	  在 Mlytics，我設計了一個 分散式 P2P signaling infrastructure，
	  支援 100K concurrent connections 和 400K messages per second。
	  架構設計：使用 Centrifugo cluster 作為 WebSocket gateway 處理大量連線，
	  	多台 Golang backend servers 處理業務邏輯和 peer selection，
	      Redis Cluster 維護 peer scoring 和 ranking。
	  這是一個完全 stateless 的設計，可以 horizontal scaling，並確保 high availability。
	  ```
- ### 1. Centrifugo Layer（Connection Layer）
	- **職責：**
		- WebSocket connection management
		- Message pub/sub
		- Connection presence tracking
		- Automatic reconnection handling
		- Message history
	- **為什麼是 Cluster：**
		- 單台 Centrifugo 處理 ~30-40K connections
		- 需要 3+ 台達到 100K
		- Horizontal scaling - 可以動態增加節點
		- High availability - 一台掛了不影響其他
	- **Centrifugo 之間如何協調：**
		- 透過 Redis 做 pub/sub
		- 共享 presence information
		- 共享 message history
- ### 2. Backend Layer（Business Logic Layer）
	- **職責：**
		- Peer registration 和 lifecycle management
		- Peer scoring algorithm
		- Peer selection logic
		- Signaling message routing
		- Metrics collection
	- **為什麼也要多台：**
		- **Stateless design** - 任何 backend 都可以處理任何請求
		- Load balancing - 分散 CPU 負載
		- Fault tolerance - 一台掛了其他繼續服務
		- Scalability - 根據負載動態增減
	- **Backend 之間如何協調：**
		- 不需要直接溝通
		- 所有 state 都在 Redis
		- 透過 Redis Pub/Sub 做 event notification
- ### 3. Redis Cluster（State Layer）
	- **為什麼用 Cluster：**
		- 單台 Redis 有記憶體限制
		- 100K peers 的 state 需要大量記憶體
		- High availability（master-slave replication）
		- Data sharding（自動分散資料）
- ## 為什麼這樣設計能處理 400K msg/sec
	- ### 職責分離
	  
	  ```
	  Centrifugo Layer:
	  - 只負責 WebSocket I/O
	  - 高度優化的 connection handling
	  - 用 Golang 寫的，原生支援高併發
	  
	  Backend Layer:
	  - 只負責業務邏輯
	  - Stateless，可以平行處理
	  - 不被 I/O blocking
	  
	  Redis Layer:
	  - 高效能的 in-memory 操作
	  - Pipelined 批次操作
	  - Cluster 分散負載
	  ```
	- ### Horizontal Scaling
	  
	  ```
	  遇到效能瓶頸時：
	  
	  - Centrifugo 慢？
	  → 加 Centrifugo nodes
	  
	  - Backend 慢？
	  → 加 Backend nodes
	  
	  - Redis 慢？
	  → 加 Redis shards
	  
	  各層獨立 scale，不互相影響
	  ```
	- ### 批次處理
	  
	  ```
	  不是每個 message 都要：
	  - 立即寫 DB
	  - 立即計算 score
	  - 立即通知所有人
	  
	  而是：
	  - Message 先在 Centrifugo buffer
	  - Backend 批次處理（每 100ms）
	  - Redis 用 pipeline 批次寫入
	  - Score 定期批次更新（每 10s）
	  ```
	-