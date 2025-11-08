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
- ```
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
- ```
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
  
  **方案 1: 使用 Completer 確保順序**
  
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
- ### 關鍵學習
  
  **教訓：**
- **非同步操作的順序很重要** - WebRTC 建立連線需要時間
- **不能假設對方已經 ready** - 需要明確的握手機制
- **Lock 很重要** - 避免 concurrent 操作造成 inconsistent state