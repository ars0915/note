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
	-
	-
-