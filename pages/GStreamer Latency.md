public:: true
tags:: Multicast, GStreamer

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
	  - 否則 audio source 會 underrun
	  ```
- ## Latency Query 機制
	- **`min-latency`**：pipeline 中的最小延遲，意味著同步到時鐘的下游元素必須等待的最小時間，以確保收到當前運行時間的所有數據
	- **`max-latency`**：pipeline 中的最大延遲，意味著同步到時鐘的元素允許等待接收當前運行時間所有數據的最大時間
	-