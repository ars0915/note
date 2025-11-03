- **Redis 資料結構與使用場景：**
  ```
  String  → 簡單 key-value、計數器、session
  Hash    → 物件儲存（user:{id}:name, user:{id}:level）
  List    → 佇列、最新消息列表
  Set     → 去重、共同好友
  ZSet    → 排行榜（score 排序）
  ```
- **快取三大問題（面試必考）：**
	- 1. **快取穿透（Cache Penetration）**
		- 問題：查詢不存在的資料，繞過快取直擊 DB
		- 解法：
			- 布隆過濾器（Bloom Filter）
			- 快取空值（設短過期時間）
	- 2. **快取擊穿（Cache Breakdown）**
		- 問題：熱點 key 過期，瞬間大量請求打到 DB
		- 解法：
			- 互斥鎖（singleflight）
			- 永不過期 + 非同步更新
	- 3. **快取雪崩（Cache Avalanche）**
		- 問題：大量 key 同時過期
		- 解法：
			- 過期時間加隨機值
			- 多層快取
			- 熔斷降級
- **Redis 持久化：**
	- RDB：快照，fork 子進程，適合備份
	- AOF：記錄每個寫命令，更安全但檔案大
- **面試陷阱題：「Redis 是單執行緒為什麼還這麼快？」**
	- IO 多路複用（epoll）
	- 純記憶體操作
	- 高效的資料結構
	- 單執行緒避免 context switch