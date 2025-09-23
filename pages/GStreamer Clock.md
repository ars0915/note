public:: true
tags:: GStreamer

- GStreamer 透過 `GstClock` object , buffer timestamp, `SEGMENT` event 去同步 pipeline 的 streams
- # Clock running-time
	- 因為電腦會有多個時間來源，所以 GStreamer 有很多 `GstClock`
	- **absolute-time** 可以透過 `gst_clock_get_time ()`取得
	- **running-time** 是由 `absolute-time - base-time` 算出
	- **base-time** 是當 GStreamer 的 pipeline 切換到 PLAYING 狀態時，選擇一個 GstClock（通常是系統時鐘或 audio/video sink 所提供的時鐘）。
		- 在 PAUSED 狀態下，running-time 是靜止的（即便時鐘還在走）。
		- 每次重新進入 PLAYING 時，pipeline 會更新 base-time，使 running-time 是連續的。
		- 這樣可以確保時間同步正確，並避免例如 seek 或暫停後時間錯亂的情況。
- # Buffer running-time
	- 在 GStreamer 裡，每個 buffer 都有一個 timestamp，但這是 stream-time，不是 running-time。
	  要計算出 running-time，需要使用 `SEGMENT` 事件中攜帶的資訊。可以透過 GStreamer 的 API gst_segment_to_running_time()取得
	- ## 為什麼要知道 buffer 的 running-time？
		- 用來同步播放
		  pipeline 裡所有元件（尤其是 sink）都會根據 clock 的當前時間減掉 base-time 得到「現在的 running-time」
			- 如果 buffer 的 running-time 小於這個值，表示它「應該已經播過了」。
			- 如果大於，表示「還沒到播放時間，要等一下」。
		- 實際播放時間 = buffer.running_time + pipeline.latency
		  Sink 元件會用這個計算來決定 buffer 要不要丟掉、延遲或立即播放。
	- ## 非即時 (non-live) source 的行為
		- 從檔案讀取影片的情境，第一個 buffer 通常 timestamp 是 0。
		- 若使用者做了 flushing seek（例如跳到 30 秒位置），新的 SEGMENT 會重新定義時間區段，buffer running-time 會從 0 開始。
	- ## 即時 (live) source 的行為
		- 把 capture 時刻對應的 pipeline running-time 設為 buffer 的 timestamp。
		  這樣 playback 才能與現實世界對齊（例如直播畫面必須延遲固定幾百 ms，但不能隨便亂跳時間）。
	-
	-
	-
	-