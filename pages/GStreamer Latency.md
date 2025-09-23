public:: true
tags:: GStreamer

- ## 什麼是 Latency
	- latency 是指一個在時間戳 0 被捕獲的樣本到達 sink 所需的時間，這個時間是相對於 pipeline 的時鐘測量的。
	- 這是 處理延遲，不是網路延遲或播放延遲。
- ## 為什麼需要 Latency 補償？
	- ### Live Source 的時間問題：
	  ```
	  Audio Source 範例：
	  - T=0: 開始捕獲第一個樣本
	  - T=1s: 收集完整個 buffer（44100 samples @ 44100Hz）
	  - Buffer timestamp = 0，但現在 clock >= 1s
	  - Sink 判斷：這個 buffer 太晚了！→ 丟棄
	  
	  沒有 latency 補償 = 所有 buffer 都會被丟棄！
	  ```
- ## Latency Compensation（延遲補償）
  用來同步播放 & 避免 drop 掉剛進來但 timestamp 很早的 buffer
	- 計算 latency：
		- 在進入 PLAYING 之前
		- 向所有 sink 發出 LATENCY QUERY
		- 回傳每條 path 的 latency 值（由 element 報告）
		- pipeline 取 最大 latency
	- 廣播 LATENCY event：
		- 將這個 latency 值透過 LATENCY 事件廣播給所有 element（尤其是 sink）
	- sink 等待播放：
		- 每個 sink 都延後播放這個時間
		- 這樣所有 sink 播放時間同步
- ## 複雜場景下的 Latency 挑戰
	- 音視頻同步 capture
	  ```
	  假設：
	  - Audio source latency = 20ms  
	  - Video source latency = 33ms
	  
	  解決方案：
	  - 總 latency 必須 >= 33ms（取最大值）
	  - Audio 需要額外 13ms 緩衝（33-20=13ms）
	  - 否則 audio source 會 underrun (音頻設備要播放時，發現緩衝區是空的)
	  ```
- ## Latency Query 機制
	- **`min-latency`**：pipeline 中的最小延遲，下游必須延後多久，才能保證 upstream 的資料來得及
	  ```
	  假設：
	  	•	base-time = 0
	  	•	clock-time = 10.0 秒 → running-time = 10.0 秒
	  	•	sink 準備播放 timestamp = 10.0 的音訊資料
	  
	  min-latency = 100ms
	  
	  ```
	- **`max-latency`**：pipeline 中的最大延遲，最多能等多久來接收一個特定時間點的資料
	  ```
	  假設：
	  - sink 的 running-time 現在為 10.0 秒
	  - 它想播放 timestamp = 10.0 的音訊 buffer
	  
	  情況 1：max-latency = 100ms
	  → sink 最多等到 clock-time = base-time + 10.1 秒
	  → 如果 buffer 還沒來，就 drop 或播放空白
	  
	  情況 2：max-latency = 500ms
	  → sink 最多等到 clock-time = base-time + 10.5 秒
	  → 有更多緩衝時間等待這個 buffer
	  ```
	- 計算規則
	  ```c
	  // 對於非 leaky buffering 元素：
	  min_latency = upstream_min_latency + own_min_latency;
	  
	  if (upstream_max_latency == NONE || own_max_latency == NONE)
	      max_latency = NONE;
	  else
	      max_latency = upstream_max_latency + own_max_latency;
	  
	  // 對於 leaky buffering 元素（如 audio sink）：
	  // Leaky buffer 不會讓數據累積，它會主動丟棄，所以總延遲受限於最小的那個環節
	  max_latency = MIN(upstream_max_latency, own_max_latency);
	  ```
- ## 全域 Latency 計算
  ```
  // Pipeline 收集所有 sink 的 latency 資訊：
  latency = MAX(all min latencies);  // 取最大的最小延遲
  
  // 檢查是否可行：
  if (MIN(all max latencies) < latency) {
      // 不可能的情況！需要增加 buffering
      error("Pipeline cannot be played");
  }
  ```
	- example
		- 範例 1
		  ```
		  sink1: [20-20ms], sink2: [33-40ms]
		  MAX(20, 33) = 33ms						// 全域最小需要 33ms
		  MIN(20, 40) = 20ms < 33ms → 不可能！❌	// 全域最大只能 20ms
		  ```
		  Pipeline 需要：至少 33ms 延遲才能正常運作
		  Pipeline 限制：最多只能承受 20ms 延遲
		  → 這個 pipeline 無法同時滿足所有元素的需求
		  **解決方法**：在 chain 中加入 queue 來增加 buffering。
		- 範例 2
		  ```
		  sink1: [20-50ms], sink2: [33-40ms]  
		  MAX(20, 33) = 33ms
		  MIN(50, 40) = 40ms >= 33ms → latency = 33ms ✅
		  ```
- ## Dynamic Latency
	- 什麼情況會改變 latency：
		- 新增編碼器、轉碼器
		- 調整 properties（例如 buffer-size）
		- 切換 source
	- 怎麼應對：
		- 某個 element 發出 LATENCY bus message
		- 應用程式（App）收到後 可以選擇：
			- 重新做 latency query
			- 廣播新的 LATENCY event
	- latency 重算可能導致聲音／畫面「跳動」或「短暫卡頓」
	  所以通常只會在「允許中斷」的情境下（例如使用者暫停時）去更新 latency。