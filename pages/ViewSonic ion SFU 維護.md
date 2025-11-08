- 我解決了幾個關鍵的 production 問題，包括：
	- **Race condition** 導致 signaling 失敗
	- **斷線重連**機制的設計
	- **黑屏檢測**和自動修復
	- **MediaProjection 被 release** 的處理
- ## 案例 1: Signaling Race Condition
	- ### 問題描述
		- **現象：**
		  
		  ```
		  場景：Sender 要開始 screen sharing
		  問題：Sender 發送的 signal messages 沒有傳到 ionSFU server
		  
		  用戶體驗：
		  - Sender 按下「開始投屏」
		  - Receiver 端沒有畫面
		  - 沒有任何錯誤訊息
		  ```
		- **根本原因：**
		  ```
		  時序問題：
		  - 正常流程應該是：
		  1. Receiver 建立 ionSFU peer connection
		  2. Receiver 註冊 signal handler
		  3. Sender 發送 signal messages
		  4. Receiver 收到並處理
		  - 實際發生的：
		  1. Sender 發送 joinDisplay
		  2. Sender 立即發送 start remote screen message
		  3. Receiver 的 ionSFU peer 還沒建立完成
		  4. Receiver 的 signal handler 還沒註冊
		  5. ❌ Sender 的訊息全部丟失
		  ```
		- **技術細節：**
		  ```
		  Sender side (發送端):
		  joinDisplay() 
		  → 立即 sendRemoteScreenInfo()  ← 太快了！
		  
		  Receiver side (接收端):
		  收到 joinDisplay
		  → 開始建立 ionSFU connection (非同步)
		    → 建立 peer connection
		    → 註冊 signal handler  ← 還沒完成
		    
		  結果：message 在 handler 註冊前就到了
		  ```
		- ### 解決方案
			- **方案 1: 使用 Completer 確保順序**
			  
			  ```
			  概念：
			  - Receiver 建立好 connector 後才通知 Sender
			  - Sender 收到確認後才發送 remote screen info
			  
			  實現：
			  Receiver:
			  1. 收到 joinDisplay
			  2. 建立 ionSFU connection
			  3. 註冊 signal handler
			  4. 回傳 "ready" 訊息給 Sender ✅
			  
			  Sender:
			  1. 發送 joinDisplay
			  2. 等待 Receiver 的 "ready" 訊息
			  3. 收到後才發送 start remote screen ✅
			  ```
			  
			  **方案 2: 調整流程設計**
			  ```
			  原本：
			  Sender → joinDisplay → 立即 startRemoteScreen
			  
			  改成：
			  Sender → joinDisplay
			      → 等待 Receiver 回覆
			      → 收到確認後才 startRemoteScreen
			  ```
			  
			  **方案 3: 處理重連時的 Race Condition**
			  ```
			  問題：Receiver 斷線重連時也會有同樣問題
			  
			  解決：
			  - 重建 ionSFU client 時使用 lock
			  - 確保「stop 後不重建」的邏輯
			  - 避免重連和 stop 同時發生的 race condition
			  ```
- ## 案例 2: 斷線自動重連設計
	- ### 問題背景
		- **場景：大傳大（一對多）**
		  
		  ```
		  架構：
		  Host (1) → ionSFU Server → Members (N)
		  
		  連線：
		  - Host 透過 WebSocket 連到 signaling server
		  - Members 透過 WebSocket 連到 signaling server
		  - 透過 ionSFU 做 media forwarding
		  ```
		  
		  **問題：**
		  ```
		  1. Host 網路斷線
		   → Members 沒畫面
		   → 需要 Host 自動重連
		  
		  2. Member 斷線
		   → Host 不知道
		   → 浪費資源繼續傳送
		  
		  3. 兩邊同時斷線重連
		   → 可能建立多個重複連線
		   → State 不一致
		  ```
	- ### 解決方案
		- #### Scenario 1: Host 斷線重連
		  **設計：**
		  ```
		  偵測斷線：
		  - WebSocket connection lost
		  - 或 ionSFU peer connection failed
		  
		  處理流程：
		  1. Host 檢測到斷線
		  2. Host 嘗試重新建立 WebSocket
		  3. Host 檢查是否有 existing DisplayGroupSession
		   - 如果有 → 清理舊的 session
		   - 建立新的 session
		  4. 重新建立 ionSFU connection
		  5. 通知所有 Members 重新連線
		  ```
		  
		  **關鍵考量：**
		  ```
		  避免重複連線：
		  - Host 重連時帶上 session ID
		  - Server 檢查 session ID
		  - 存在 → 清理舊的
		  - 不存在 → 建立新的
		  
		  狀態同步：
		  - Host 重連後需要知道哪些 Members 還在
		  - Members 需要知道 Host 已經重連
		  - 使用 session version 避免舊訊息
		  ```
		- ## Scenario 2: Member 已斷線但 Host 重連
- **問題：**
  
  ```
  時序：
  1. Member 斷線（Host 不知道）
  2. Host 也斷線
  3. Host 重連成功
  4. Host 發送訊息給所有 "在線" 的 Members
  5. Member 這時候才重連
  6. ❌ Member 收到的是舊 session 的訊息
  ```
  
  **解決：**
  
  ```
  Member side:
  1. Member 檢測到自己已經斷線
  2. Member 重連時告訴 Host：
   "我之前斷線了，請重新初始化"
   
  3. Host 收到這個訊息
   → 清理舊 session
   → 建立新 session
   → 重新發送初始化訊息
  
  避免：
  - 不要讓 Member 處理舊 session 的訊息
  - Session ID mismatch → 要求 Host 重建
  ```
  
  **實現重點：**
  
  ```
  Session Version Control:
  - 每次重連都有新的 session ID
  - 所有訊息都帶 session ID
  - 收到不匹配的 session ID → 忽略或要求重連
  
  Lock Management:
  - 重連過程要加鎖
  - 避免多個 goroutines 同時重連
  - 確保 state transition 是 atomic
  
  Timeout Handling:
  - 設定 reconnection timeout
  - 超時後放棄重連，通知用戶
  ```