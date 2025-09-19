public:: true
tags:: GStreamer

- ## 什麼時 Latency
	- latency 是指一個在時間戳 0 被捕獲的樣本到達 sink 所需的時間，這個時間是相對於 pipeline 的時鐘測量的
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
	- **`min-latency`**：pipeline 中的最小延遲，意味著同步到時鐘的下游元素必須等待的最小時間，以確保收到當前運行時間的所有數據
	- **`max-latency`**：pipeline 中的最大延遲，意味著同步到時鐘的元素允許等待接收當前運行時間所有數據的最大時間
	  ```
	  假設現在時鐘 = 10.0 秒，要播放這個時間點的音頻
	  
	  Max-latency = 100ms 意思是：
	  "我最多等到 10.1 秒，如果到時候還沒收到 10.0 秒的音頻數據，
	  我就放棄了（可能播靜音或重複前一個 sample）"
	  
	  Max-latency = 500ms 意思是：
	  "我可以等到 10.5 秒，給更多時間讓 10.0 秒的數據到達"
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
-
-