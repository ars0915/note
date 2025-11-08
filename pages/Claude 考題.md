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
	-
-