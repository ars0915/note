## Redis 資料結構與使用場景：
```
String  → 簡單 key-value、計數器、session
Hash    → 物件儲存（user:{id}:name, user:{id}:level）
List    → 佇列、最新消息列表
Set     → 去重、共同好友
ZSet    → 排行榜（score 排序）
```
- ## 快取三大問題（面試必考）：
	- 1. **快取穿透（Cache Penetration）**
		- 問題：查詢不存在的資料，繞過快取直擊 DB
		- 解法：
			- 布隆過濾器（Bloom Filter）
			  ```go
			  // 初始化時，把所有存在的 user_id 加入布隆過濾器
			  func Init() {
			      bf := bloom.New(1000000, 5) // 100萬個元素，5個hash函數
			      
			      // 從DB讀取所有user_id
			      userIDs := db.GetAllUserIDs()
			      for _, id := range userIDs {
			          bf.Add([]byte(id))
			      }
			  }
			  
			  func GetUser(userID string) (*User, error) {
			      // 1. 先用布隆過濾器檢查
			      if !bf.Test([]byte(userID)) {
			          return nil, errors.New("user not found")  // 直接返回，不查DB
			      }
			      
			      // 2. 查 Redis
			      user, err := redis.Get("user:" + userID)
			      if err == nil {
			          return user, nil
			      }
			      
			      // 3. 查 DB
			      user, err = db.GetUser(userID)
			      if err != nil {
			          return nil, err
			      }
			      
			      // 4. 寫入 Cache
			      redis.Set("user:" + userID, user, 10*time.Minute)
			      return user, nil
			  }
			  ```
				- **Singleflight 如何運作：**
				  ```
				  Request 1: Cache miss → 開始查DB
				  Request 2: Cache miss → 等待 Request 1 的結果
				  Request 3: Cache miss → 等待 Request 1 的結果
				  ...
				  Request 1: 查DB完成 → 所有請求共享這個結果
				  ```
			- 快取空值（設短過期時間）
			  ```go
			  func GetUser(userID string) (*User, error) {
			      // 1. 查 Redis
			      value, err := redis.Get("user:" + userID)
			      if err == nil {
			          if value == "null" {  // 之前查過，不存在
			              return nil, errors.New("user not found")
			          }
			          return parseUser(value), nil
			      }
			      
			      // 2. 查 DB
			      user, err := db.GetUser(userID)
			      if err != nil {
			          // DB 也沒有，快取空值
			          redis.Set("user:" + userID, "null", 5*time.Minute)  // 短過期時間
			          return nil, errors.New("user not found")
			      }
			      
			      // 3. 寫入正常值
			      redis.Set("user:" + userID, user, 30*time.Minute)
			      return user, nil
			  }
			  ```
	- 2. **快取擊穿（Cache Breakdown）**
		- 問題：熱點 key 過期，瞬間大量請求打到 DB
		- 解法：
	- 3. **快取雪崩（Cache Avalanche）**
		- 問題：大量 key 同時過期
		- 解法：
			- 過期時間加隨機值
			- 多層快取
			- 熔斷降級
- ## Redis 持久化：
	- RDB：快照，fork 子進程，適合備份
	- AOF：記錄每個寫命令，更安全但檔案大
- ## 面試陷阱題：「Redis 是單執行緒為什麼還這麼快？」
	- IO 多路複用（epoll）
	- 純記憶體操作
	- 高效的資料結構
	- 單執行緒避免 context switch
- ## 分散式快取的一致性問題
	- 這是**實務問題**，focus 在：
	  1. **Cache 和 DB 的一致性**
	  2. **多個 Cache 節點的一致性**
	- ### Cache 和 DB 的一致性
		- **常見問題：**
		  ```
		  Thread 1: 更新 DB → 刪除 Cache
		  Thread 2: 讀 Cache miss → 讀舊的 DB → 寫入 Cache
		  → Cache 裡是舊資料！
		  ```
		- 策略：
			- **Cache Aside (最常用)** 記得讀 DB 時加鎖
			  ```go
			  func Get(key string) (string, error) {
			      // 1. 先讀 cache
			      value, err := redis.Get(key)
			      if err == nil {
			          return value, nil
			      }
			      
			      // 2. Cache miss，加鎖
			      lockKey := "lock:" + key
			      lock := redis.SetNX(lockKey, "1", 10*time.Second)
			      if !lock {
			          // 沒搶到鎖，等一下重試
			          time.Sleep(50 * time.Millisecond)
			          return Get(key)  // 重試
			      }
			      defer redis.Del(lockKey)
			      
			      // 3. Double check（可能其他 thread 已經寫入了）
			      value, err = redis.Get(key)
			      if err == nil {
			          return value, nil
			      }
			      
			      // 4. 讀 DB
			      value, err = db.Get(key)
			      if err != nil {
			          return "", err
			      }
			      
			      // 5. 寫 Cache
			      redis.Set(key, value, 10*time.Minute)
			      
			      return value, nil
			  }
			  
			  func Update(key, value string) error {
			      // 1. 加鎖（防止讀操作同時進行）
			      lockKey := "lock:" + key
			      for !redis.SetNX(lockKey, "1", 10*time.Second) {
			          time.Sleep(10 * time.Millisecond)
			      }
			      defer redis.Del(lockKey)
			      
			      // 2. 更新 DB
			      err := db.Update(key, value)
			      if err != nil {
			          return err
			      }
			      
			      // 3. 刪除 Cache
			      redis.Del(key)
			      
			      return nil
			  }
			  ```
				- **為什麼刪除而不是更新 Cache？**
					- 更新可能失敗，導致不一致
					- 更新複雜（如果有計算邏輯）
					- 刪除簡單，下次讀取時自然更新
			- **Read/Write Through（較少用）**
			  ```
			  應用層 ←→ Cache ←→ DB
			           (Cache 負責同步)
			  ```
			- **Write Behind（非同步，最終一致性）**
			  ```
			  寫入 Cache → 回傳成功
			              ↓
			           (非同步)
			              ↓
			            寫入 DB
			  ```
		- | 策略 | 讀 | 寫 | 一致性 | 效能 | 複雜度 |
		  | ---- | ---- | ---- |
		  | **Cache Aside** | App 自己讀 | 更新 DB + 刪 Cache | 最終一致 | 好 | 低 |
		  | **Read Through** | Cache 代理讀 | App 自己寫 | 最終一致 | 好 | 中 |
		  | **Write Through** | Cache 代理讀 | Cache 同步寫 | 強一致 | 寫慢 | 中 |
		  | **Write Behind** | Cache 代理讀 | Cache 非同步寫 | 最終一致 | 極快 | 高 |
	- ### 多個 Cache 節點的一致性
		- **一致性 Hash（Consistent Hashing）：**
		  ```
		  Problem: 普通 Hash
		  node = hash(key) % 3  *// 3 個節點*
		  
		  - 增加一個節點變 4 個
		    → 大量 key 要重新分配 → Cache 全部失效！  
		  - Solution: 一致性 Hash
		    → 增加節點只影響相鄰節點  
		    → 大部分 key 不需要重新分配
		  ```
	-