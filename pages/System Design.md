public:: true

- 建議準備這些場景：
  
  **遊戲匹配系統：**
  ```
  1. 玩家加入匹配池（Redis ZSet，score = MMR）
  2. Matching Service 定期掃描
  3. 組隊邏輯（相近 MMR）
  4. 創建房間
  5. 通知玩家（WebSocket）
  
  考量：
  - 如何避免重複匹配？
  - MMR 範圍如何動態調整？
  - 高峰期如何擴展？
  ```
  
  **排行榜系統：**
  ```
  Redis ZSet 是經典解法
  
  ZADD leaderboard 1000 "player1"  // O(log N)
  ZREVRANGE leaderboard 0 99       // Top 100
  ZREVRANK leaderboard "player1"    // 查排名
  
  考量：
  - 分區排行榜（週榜、月榜、全服榜）
  - 定期歸檔到 DB
  - 防作弊
  ```
  
  **聊天系統：**
  ```
  1. 客戶端 WebSocket 連接
  2. 訊息發送 → Gateway → MQ → Message Service
  3. Message Service 推送給在線用戶
  4. 離線消息存 DB
  
  考量：
  - 訊息順序保證（partition key = chat_room_id）
  - 歷史訊息查詢（DB + Cache）
  - 已讀未讀狀態
  ```
-