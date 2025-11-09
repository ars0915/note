# DB & Cache
	- ## Q: 你在設計一個遊戲的排行榜系統，需要即時更新玩家分數並顯示 Top 100。你會選擇 Redis 的哪種資料結構？為什麼？
	- A: 用 ZSET 計分數和排序
	- ## Q: 如果我現在要實現以下功能，你會用什麼 Redis 命令？
		- 更新玩家 `player:123` 的分數為 9527
		- 查詢 Top 10 玩家（從高到低）
		- 查詢玩家 `player:123` 的排名
		- 查詢分數在 8000-10000 之間的玩家數量
	- A:
		- ZADD leaderboard 9527 player:123
		- ZREVRANGE leaderboard 0 9 (`ZREVRANGE leaderboard 0 10` - 會回傳 **11 個元素**（0, 1, 2...10）)
		- ZREVRANK leaderboard player:123 (`ZSCORE` 會回傳分數)
		- ZCOUNT leaderboard 8000 10000
	- ## Q: 你的排行榜系統上線後，發現有個問題：
		- 排行榜有 100 萬玩家
		  每秒有 10,000 次查詢 Top 100
		  Redis 開始出現效能問題
		- **問題**：
			- 為什麼 `ZREVRANGE` 在大量查詢時會變慢？
			- 你會怎麼優化這個場景？（提示：結合你文件中的「快取三大問題」）
	- A:
		- **ZSET 不需要每次重新排序**
		- ZSET 底層用 **skip list**（跳表）實現，本身就是有序的
		- `ZREVRANGE` 的時間複雜度是 **O(log N + M)**
			- log N：定位到起始位置
			- M：返回的元素數量（這裡是 100）
		- 在 100 萬玩家的情況下，單次查詢不算慢
		- **問題在於：每秒 10,000 次查詢 × O(log N + 100) = 壓力太大**
		- 優化方法可以 **先在應用層做一層快取**
	- ## Q: Top 100 是個熱點數據，具體怎麼快取？會遇到什麼問題？
		- 這是你文件中的「**快取擊穿**」問題
		- 如果 Top 100 的快取過期了，會發生什麼？
		- 怎麼避免瞬間 10,000 個請求同時打到 Redis？
	- A:
		- 使用互斥鎖的方式拿資料 -> 這樣就把 10,000 QPS 降到了 1 QPS（對 Redis 的壓力）！
		- **快取擊穿（Cache Breakdown）**：**單一熱點 key** 過期
			- 例如：Top 100 這個 key 過期
			- 瞬間 10,000 個請求打到 Redis
		- **快取雪崩（Cache Avalanche）**：**大量不同的 key** 同時過期
			- 例如：100 萬個玩家資料同時在晚上 12 點過期
			- 解法才是：隨機過期時間、多層快取
	- ## Q: 資料庫事務題
		- **你在開發一個遊戲商城，玩家購買道具的流程是：**
			- 檢查玩家金幣是否足夠
			- 扣除金幣
			- 新增道具到背包
		- **問題：**
			- 這需要什麼隔離級別？為什麼？
			- 如果用 `Read Committed`，會發生什麼問題？
			- 怎麼用樂觀鎖實現這個場景？（寫出關鍵的 SQL）
	- A：
		- 隔離級別 Repeatable Read + SELECT FOR UPDATE
		- 如果用 Read Committed 會拿到其他筆交易扣款前的金額而可能扣到負數
		- 樂觀鎖 **UPDATE 時檢查 version**：
		  ```sql
		  -- 步驟 1: 讀取（不加鎖）
		  SELECT id, gold, version FROM users WHERE id = 123;
		  -- 假設讀到：gold=500, version=10
		  
		  -- 步驟 2: 業務邏輯檢查
		  IF gold >= 100 THEN
		      -- 步驟 3: 更新時檢查 version（關鍵！）
		      UPDATE users 
		      SET gold = gold - 100, 
		          version = version + 1
		      WHERE id = 123 AND version = 10;  -- ← 關鍵：確保沒被改過
		      
		      -- 步驟 4: 檢查影響行數
		      IF affected_rows == 0 THEN
		          -- version 不對，表示被別人改過了，重試！
		          ROLLBACK;
		          RETRY;
		      ELSE
		          INSERT INTO inventory (user_id, item_id) VALUES (123, 456);
		          COMMIT;
		      END IF;
		  END IF;
		  ```
	- ## Q: 樂觀鎖 vs 悲觀鎖，什麼時候用哪個？
		- 提示：想想「衝突機率」和「重試成本」
	- A:
		- **關鍵是「衝突機率」，不是「流量大小」**
			- 樂觀鎖：適合「讀多寫少、衝突少」
				- 為什麼可以用樂觀鎖？
					- 衝突機率低（我改我的資料，你改你的資料）
					- 即使重試，成本也不高
			- 悲觀鎖：適合「寫多、衝突多」
				- 為什麼要用悲觀鎖？
					- 衝突機率極高（1000人搶同一個商品）
					- 用樂觀鎖會導致 990 人重試 → 浪費資源
					- 不如直接排隊（悲觀鎖）
		- **重試成本 = 失敗後重新執行的代價**
			- 樂觀鎖的重試流程：
			  ```
			  第1次嘗試：讀DB(10ms) + 業務邏輯(5ms) + 寫DB失敗(5ms) = 20ms
			  第2次嘗試：讀DB(10ms) + 業務邏輯(5ms) + 寫DB失敗(5ms) = 20ms
			  第3次嘗試：讀DB(10ms) + 業務邏輯(5ms) + 寫DB成功(5ms) = 20ms
			  總耗時：60ms
			  
			  如果衝突率 90%，可能要重試 10 次才成功 → 200ms！
			  ```
			- 悲觀鎖的等待：
			  ```
			  排隊等待：100ms
			  執行：20ms
			  總耗時：120ms（一次成功，不重試）
			  ```
			- 在高衝突場景下，悲觀鎖反而更快！
	- ## Q: 你的遊戲有個「每日簽到」功能，100萬玩家每天早上8點同時簽到。
		- 你會用什麼資料結構儲存「今天已簽到的玩家」？（Redis）
		- 如何避免快取雪崩？（100萬玩家的簽到記錄都在晚上12點過期）
			- ```
			  1. 玩家登入時要顯示：
			     - "你今天還沒簽到" 
			     - "你已連續簽到 5 天"
			     
			  2. 排行榜：
			     - "連續簽到天數 TOP 100"
			     
			  3. 獎勵系統：
			     - "連續簽到 7 天送鑽石"
			  ```
			  
			  **現在問題來了：**
				- 如果 100 萬玩家的「連續簽到天數」都快取在 Redis
				- 晚上 12 點全部過期
				- 隔天早上 8 點，100 萬玩家登入
				- 都要從 DB 讀取 → DB 爆炸 💥
		- 如果要查詢「連續簽到7天的玩家」，你會怎麼設計？
	- A:
		- 只看今天已簽到的玩家，可以用 Set 存已經簽到的人
		- 避免快取雪崩
			- ```
			  // 正確示範：加上隨機值（0-1小時）
			  randomTTL := 24*time.Hour + time.Duration(rand.Intn(3600))*time.Second
			  redis.Set("user:123:streak", "5", randomTTL)
			  ```
			- 為什麼有效？
				- 原本：100萬個key在 00:00:00 同時過期
				- 現在：100萬個key在 00:00:00 ~ 01:00:00 之間分散過期
				- DB 壓力分散到 1 小時內
		- 查詢「連續簽到7天」
			- 方案 A：String 存連續天數（簡單）
			  ```
			  Key: user:{id}:streak
			  Value: {last_checkin_date}:{streak_days}
			  ```
				- **優點：** 簡單，查詢快
				- **缺點：** 只知道連續天數，不知道具體哪天簽到了
			- 方案 B：Bitmap 存每天的簽到狀態（彈性）
			  ```
			  Key: user:{id}:checkin:{year}
			  Value: Bitmap（365 bits，每個 bit 代表一天）
			  
			  例如：
			  user:123:checkin:2024
			  Bit 0 = 1月1日 (1 = 已簽到)
			  Bit 1 = 1月2日 (0 = 未簽到)
			  ...
			  Bit 306 = 11月3日 (1 = 已簽到)
			  ```
				- **優點：**
					- 節省空間（365天只需 46 bytes）
					- 可以查詢「這個月簽到了幾天」
					- 可以補簽
				- **缺點：** 計算連續天數需要多次 GetBit
			- | 需求 | 方案 A (String) | 方案 B (Bitmap) |
			  |---|---|---|
			  | 只需連續天數 | ✅ 推薦 | ❌ 太複雜 |
			  | 需要查詢歷史 | ❌ 做不到 | ✅ 推薦 |
			  | 需要補簽功能 | ❌ 做不到 | ✅ 推薦 |
			  | 空間效率 | 普通 | ✅ 極高 |
	- ## Q: 如果要做「連續簽到排行榜」呢？
	- A:
		- 需要再加一個 ZSet
		  ```
		  Key: checkin:leaderboard
		  Value: ZSet {member: userID, score: streakDays}
		  
		  ZADD checkin:leaderboard 7 user:123
		  ZADD checkin:leaderboard 10 user:456
		  
		  查詢 Top 100：
		  ZREVRANGE checkin:leaderboard 0 99 WITHSCORES
		  ```
		- | 需求 | 方案 | Redis 命令 |
		  | ---- | ---- | ---- |
		  | **「本週每天都簽到的人」** | ✅ SINTERSTORE | `SINTERSTORE result day1 day2 ... day7` |
		  | **「連續簽到 7 天的人」（任意時段）** | ❌ SINTERSTORE 做不到 | 需要 Bitmap 或 String 方案 |
		  | **「今天簽到的人」** | ✅ Set | `SMEMBERS checkin:2024-11-03` |
		  | **「連續簽到排行榜」** | ❌ Set 做不到 | 需要 ZSet |
- # MQ & Kafka
	- ## Q1: RabbitMQ 和 Kafka 的核心差異是什麼？分別適合什麼場景？
	  提示：從設計理念、訊息模型、保留機制來回答
	- A:
		- rabbitMQ 適合低延遲、保證送達和複雜路由的場景。kafka 適合高吞吐量、event sourcing、要持久化的場景。
		- RabbitMQ 是 **Push 模型**：Broker 主動推給 Consumer，消費後訊息就刪除
		- Kafka 是 **Pull 模型**：Consumer 主動拉取，訊息保留在 log 中，可以 replay
		- 這就是為什麼 Kafka 適合 event sourcing - 因為需要「重播歷史事件」。
	- ## Q2: 假設有一個 Topic 叫 user-events，有 4 個 partitions，現在有 3 個 Consumer 在同一個 Consumer Group 中消費。請問：
		- 每個 Consumer 會負責幾個 partition？
		  如果新增第 4 個 Consumer，會發生什麼事？
		  如果新增到第 5 個 Consumer，會發生什麼事？
		  這個過程中會觸發什麼機制？
	- A:
		- 每個 partition 只能被一個consumer 消費，因此會有一個 consumer 負責2個 partition，其他只負責一個。新增第四個 consumer 會讓所有 consumer 各消費一個 partition。新增第5個會有 idle consumer。在新增時會觸發 rebalance，當觸發 rebalance 時，**所有 consumer 都會停止消費**（stop-the-world），這就是為什麼 rebalance 被稱為「災難」
	- ## Q3: 你在開發一個遊戲後端的排行榜更新系統，需要確保「同一個玩家的分數更新」按照時間順序處理。
	- A:
		- 因為要按照順序處理，所以同一個玩家的分數更新要在同一個 partition 裡。這樣設計可能造成某些 partition 特別多 event。
		- 在同一個 broker 上的 partition 會互相影響
		  ```
		  Broker 1 (共享資源：CPU, Memory, Disk, Network)
		    ├─ Topic A, Partition 0
		    ├─ Topic A, Partition 1  ← 熱點！
		    ├─ Topic B, Partition 0
		    └─ Topic C, Partition 2
		  
		  這些 partitions 共享：
		    - 磁碟 I/O (寫入同一個磁碟)
		    - 網絡帶寬 (同一個網卡)
		    - CPU 資源 (處理請求)
		    - Page Cache (Linux 的文件快取)
		  ```
		- #### Dynamic Sharding 
		  概念是為高流量用戶創建獨立 topic，但僅僅創建新 topic 並不能保證資源隔離，因為 Kafka 默認會把 partitions 分散到各個 broker。
			- 要真正實現資源隔離，需要：
				- 手動指定 replica assignment，將 VIP topics 分配到專門的 brokers
				- 集群規模足夠大，能夠物理隔離不同類型的流量
				- 監控和驗證隔離效果
			- **Dynamic Sharding 的主要目的是「資源隔離」，不是「並行處理」**
				- VIP topic 通常設計為 1 個 partition
				- 重點是不讓 VIP 玩家影響普通玩家
			- **如果單個 partition 處理不了**
				- 先考慮優化 consumer（更快的機器、更高效的代碼）
				- 再考慮犧牲順序性或在應用層處理
			- **對於排行榜場景**
				- 大多數情況下，單個 partition 足夠
				- 如果真的不夠，考慮是否真的需要實時更新，還是可以批量處理
		- 在實際項目中，我會先評估集群規模。如果 brokers 數量有限（<10 個），Dynamic Sharding 的成本可能大於收益，這時候應用層限流或優化 consumer 可能是更實際的方案。
	- ## Q4: Kafka 的 ISR (In-Sync Replicas) 是什麼？請說明：
		- 什麼情況下 Follower 會被踢出 ISR？
		- `acks=all` 和 ISR 有什麼關係？
		- 如果 Leader 掛掉了，Kafka 會從哪裡選新的 Leader？
	- A:
		- 在 follower 超出時間沒像 leader fetch 或是 offset 落後太多就會被踢出。
		- 設定 acks=all 需要同步到所有 ISR follow 後才會回傳 ack。
		- Leader 掛掉會從 ISR 中選出新的 Leader
	- ## Q5: 你的團隊反應 Kafka Consumer 經常發生 rebalance，導致消息處理延遲。可能的原因有哪些？你會如何優化？
	  提示：考慮 session timeout、處理時間、Static Membership 等
	- A:
		- 最常見的 3 個原因
			- Consumer 處理時間太長
			  ```go
			  // 問題：Consumer 處理一批訊息超過 max.poll.interval.ms（預設 5 分鐘）
			  for msg := range messages {
			      processMessage(msg)  // 如果這個很慢...
			  }
			  // → Coordinator 認為 Consumer 掛了 → 觸發 rebalance
			  
			  // 解決方案：
			  props.put("max.poll.interval.ms", 600000)  // 增加到 10 分鐘
			  props.put("max.poll.records", 100)         // 減少每次拉取的數量
			  ```
			- session.timeout.ms 設定太短
			  ```go
			  // 問題：Consumer 的心跳超時
			  props.put("session.timeout.ms", 10000)  // 10 秒太短
			  
			  // 解決方案：
			  props.put("session.timeout.ms", 30000)      // 增加到 30 秒
			  props.put("heartbeat.interval.ms", 3000)    // 心跳間隔 3 秒
			  ```
			- Consumer 實例頻繁啟動/關閉
			  ```go
			  // 問題：部署、擴縮容觸發 rebalance
			  
			  // 解決方案（你提到的）：
			  props.put("group.instance.id", "consumer-1")  // Static Membership
			  ```
		- 另外，Kafka 2.4+ 可以用 Cooperative Rebalancing 減少 stop-the-world 的影響
- # Golang
	- ## Q1：以下兩個函數，哪個會造成 heap allocation？為什麼？
		- ```go
		  func foo1() int {
		      x := 42
		      return x
		  }
		  
		  func foo2() *int {
		      x := 42
		      return &x
		  }
		  ```
	- A: foo2 會用到 heap 因為 return 的是 x 的指標，在其他地方會引用到
	- ## Q2: 在高併發的 streaming 場景中，你需要頻繁處理 64KB 的 buffer。以下兩種做法哪個效能更好？為什麼？
		- ```go
		  // 方法 A
		  func handleStream1() {
		      buf := make([]byte, 64*1024)
		      // process buf...
		  }
		  
		  // 方法 B  
		  var bufferPool = sync.Pool{
		      New: func() interface{} {
		          return make([]byte, 64*1024)
		      },
		  }
		  
		  func handleStream2() {
		      buf := bufferPool.Get().([]byte)
		      defer bufferPool.Put(buf)
		      // process buf...
		  }
		  ```
	- A: 使用 sync.Pool 會比較好，減少記憶體的分配
	- ## Q3: 你在開發 WebTransport signaling server，需要管理 100,000 個並發連接的狀態。以下哪個方案更合適？
		- **方案 A：使用 sync.Map**
		  ```go
		  var connections sync.Map *// key: connID, value: *Connection*
		  ```
		- **方案 B：使用 map + RWMutex**
		  ```go
		  var (
		    mu sync.RWMutex
		    connections = make(map[string]*Connection)
		  )
		  ```
		- 請說明你的選擇理由，以及在「讀多寫少」vs「寫多讀少」場景下的考量。
	- A:
		- **sync.Map 適合：**
			- Key **只寫入一次，之後只讀取**（write-once, read-many）
			- 多個 goroutine **讀寫不同的 key**
		- 在你的 Signaling Server 場景
		  ```
		  如果：
		  - 連線建立/斷開很頻繁 → 可能 RWMutex 更好
		  - 連線建立後長時間保持，主要是查詢 → sync.Map 更好
		  ```
	- ## Q4: 解釋 `sync.RWMutex` 的 "防止 Writer Starvation" 機制。以下場景會發生什麼？
		- ```go
		  var rwmu sync.RWMutex
		  
		  // 時間軸：
		  R1.RLock() ✓ 
		  R2.RLock() ✓ 
		  W1.Lock()  ⏸ (等待 R1, R2)
		  R3.RLock() ? // 這裡會發生什麼？
		  ```
	- A: R3.RLock 會等 W1.Lock 釋放後才能讀
	- ## Q5: 以下程式會輸出什麼？有什麼問題？
		- ```go
		  func main() {
		      tasks := make(chan int, 10)
		      
		      for i := 0; i < 5; i++ {
		          go func() {
		              for task := range tasks {
		                  fmt.Println("Processing:", task)
		              }
		          }()
		      }
		      
		      for i := 0; i < 100; i++ {
		          tasks <- i
		      }
		      
		      time.Sleep(time.Second)
		  }
		  ```
	- A:
		- **沒有 close(tasks)**，worker 會永遠 block 在 `range tasks`
		- **用 `time.Sleep` 等待不可靠**，可能在 worker 完成前就退出
	- ## Q6: 在你的 Miracast streaming 場景中，需要優雅關閉系統。請設計一個使用 `context.Context` 和 `errgroup` 的方案，能夠：
		- 同時管理多個 goroutine（signaling、streaming、heartbeat）
		- 任一 goroutine 出錯時，取消其他所有 goroutine
		- 等待所有 goroutine 清理完畢
	- A:
		- errorGroup.WithContext, 把 context 傳入所有 goroutine, 當任何一個 return error 時 ctx.Err() 會傳出 error, 取消其他 goroutine
		- ```go
		  func gracefulShutdown(ctx context.Context) error {
		      g, ctx := errgroup.WithContext(ctx)
		      
		      // 啟動多個 goroutine
		      g.Go(func() error {
		          return runSignaling(ctx)  // 傳入 ctx
		      })
		      
		      g.Go(func() error {
		          return runStreaming(ctx)
		      })
		      
		      g.Go(func() error {
		          return runHeartbeat(ctx)
		      })
		      
		      // ✅ 用 g.Wait() 等待，不是 <-ctx.Done()
		      return g.Wait()  // 任一 goroutine error → 自動取消其他 → 返回第一個 error
		  }
		  
		  func runSignaling(ctx context.Context) error {
		      for {
		          select {
		          case <-ctx.Done():  // ✅ 檢查是否被取消
		              return ctx.Err()
		          default:
		              // do work...
		          }
		      }
		  }
		  ```
		  <-ctx.Done() 是在 goroutine 內部 檢查是否被取消
		  g.Wait() 是在 main 等待所有 goroutine 完成
		-
	- ## Q7: 樂觀鎖 vs 悲觀鎖 在什麼場景下你會選擇樂觀鎖（CAS + version）而不是悲觀鎖（SELECT FOR UPDATE）？
	- A: 在大量併發的場景會使用樂觀鎖，避免所有執行緒一直在等待拿鎖
	- ## Q8: 你需要實作一個 Connection Pool，要求：
		- 支援 100 個連接的並發存取
		- 連接用完要放回 pool
		- Pool 滿了就丟棄連接
		- **使用 Channel 實作比 Mutex 好在哪裡？**
		- 提示：
		  ```go
		  type Pool struct {
		    conns chan *Conn  *// buffered channel*
		  }
		  ```
	- A:
		-