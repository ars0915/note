public:: true
tags:: Miracast, GStreamer

- # RTP Jitterbuffer 的 Buffer Mode
	- ## 模式類型
	  ```cpp
	  // GStreamer 定義的模式
	  enum {
	    BUFFER_MODE_NONE = 0,     // 不使用特定模式
	    BUFFER_MODE_SLAVE = 1,    // Slave 模式 - 跟隨 pipeline clock
	    BUFFER_MODE_BUFFER = 2,   // Buffer 模式 - 自主緩衝
	    BUFFER_MODE_SYNCED = 4    // Synced 模式 - RTP 時間戳同步
	  };
	  ```
		- ### SLAVE Mode (模式 1) - 推薦用於低延遲
			- Jitterbuffer **跟隨 pipeline 的 clock**
			- 根據 pipeline 當前時間決定何時輸出封包
			- 如果封包太舊（相對於 pipeline clock），可以主動丟棄
			- **適合實時串流**，延遲較低
			  **運作方式：**
			  ```
			  Pipeline Clock: -----|-------------|-------------|----->
			                   現在時間
			  
			  Jitterbuffer:  [舊封包] [當前] [未來封包]
			                 ↓
			              可能被丟棄
			  ```
		- ### BUFFER Mode (模式 2) - 不推薦用於實時
			- Jitterbuffer **自主決定**何時輸出
			- 會盡量保持 buffer 滿，不管 pipeline clock
			- 追求**平滑播放**而非低延遲
			- 可能累積大量延遲
		- ### SYNCED Mode (模式 4)
			- 使用 **RTP 時間戳**和 **SR (Sender Report)** 同步
			- 需要 RTCP 支援
			- 精確但較複雜
	- ## 實際範例比較
		- ### 不設定 mode（預設行為）
		  
		  ```cpp
		  g_object_set(jitterbuffer,
		             "latency", 100,
		             // 沒有設定 mode
		             NULL);
		  ```
		  
		  **結果：**
			- Jitterbuffer 可能累積大量封包
			- 延遲可能達到數秒
			- 你之前看到的 500+ packets 就是這個問題
		- ### SLAVE mode（推薦）
		  
		  ```cpp
		  g_object_set(jitterbuffer,
		             "latency", 100,
		             "mode", 1,  // SLAVE mode
		             NULL);
		  ```
		  
		  **結果：**
			- Jitterbuffer 緊跟 pipeline clock
			- 太舊的封包會被標記為 "late"
			- 配合 `drop-on-latency` 可以丟棄過舊封包
			- 延遲保持在 100-200ms 左右
	- ## 為什麼你需要 SLAVE mode
		- ### 問題場景：Miracast 實時串流
		  
		  ```
		  
		  發送端 → 網路 → RTP → Jitterbuffer → Decoder → 顯示
		         (抖動)         (累積延遲？)
		  
		  ```
		- **沒有 SLAVE mode：**
		  ```
		  時間軸: 0s -------- 1s -------- 2s -------- 3s
		  封包:   [#1 #2 #3] [#4 #5 #6] [#7 #8 #9] [#10...]
		  Jitter: [#1 #2 #3 #4 #5 #6 #7 #8 #9 #10...]  ← 一直堆積
		  延遲:   100ms       500ms       1000ms      2000ms ↑
		  ```
		- **使用 SLAVE mode：**```
		  
		  - 時間軸: 0s -------- 1s -------- 2s -------- 3s
		  - 封包:   [#1 #2 #3] [#4 #5 #6] [#7 #8 #9] [#10...]
		  - Jitter: [#1 #2 #3] [#4 #5 #6] [#7 #8 #9] [#10...] ← 保持固定大小
		  - 延遲:   100ms       100ms       100ms       100ms   ← 穩定
		  - 丟棄#1      丟棄#4      丟棄#7               (如果太慢)
		  ```
-
- ## 完整的低延遲配置
  
  ```cpp
  
  // RTpbin 設定
  
  g_object_set(rtpbin_,
  
             "latency", 50,              // 目標延遲 50ms
  
             "buffer-mode", 1,           // SLAVE mode
  
             "drop-on-latency", FALSE,   // 不在 RTP 層丟（避免 broken NAL）
  
             NULL);
  
  // Jitterbuffer 設定（透過 new-jitterbuffer 信號）
  
  static void OnNewJitterBuffer(GstElement* rtpbin, GstElement* jitterbuffer, 
  
                              guint session, guint ssrc, gpointer user_data) {
  
  g_object_set(jitterbuffer,
  
               "latency", 50,            // 50ms 緩衝
  
               "mode", 1,                // SLAVE mode - 跟隨 pipeline clock
  
               "do-lost", TRUE,          // 偵測丟包
  
               "drop-on-latency", FALSE, // 不直接丟包
  
               NULL);
  
  }
  
  ```
- ## 如何驗證 SLAVE mode 是否生效
- ### 添加監控代碼
  
  ```cpp
  
  static gboolean CheckJitterBufferMode(gpointer user_data) {
  
  RtpMpegTsPlayerGst* self = static_cast<RtpMpegTsPlayerGst*>(user_data);
  
  if (!self || !self->rtpbin_) {
  
    return G_SOURCE_REMOVE;
  
  }
  
  // 獲取 jitterbuffer
  
  GValue val = G_VALUE_INIT;
  
  g_value_init(&val, GST_TYPE_ELEMENT);
  
  g_signal_emit_by_name(self->rtpbin_, "get-internal-session", 0, &val);
  
  GstElement* session = GST_ELEMENT(g_value_get_object(&val));
  
  
  
  if (session) {
  
    gint mode = 0;
  
    guint latency = 0;
  
    guint64 num_pushed = 0;
  
    guint64 num_late = 0;
  
    
  
    g_object_get(session, 
  
                 "mode", &mode,
  
                 "latency", &latency,
  
                 NULL);
  
    
  
    // 獲取統計
  
    GstStructure* stats = NULL;
  
    g_object_get(session, "stats", &stats, NULL);
  
    if (stats) {
  
      gst_structure_get_uint64(stats, "num-pushed", &num_pushed);
  
      gst_structure_get_uint64(stats, "num-late", &num_late);
  
      gst_structure_free(stats);
  
    }
  
    
  
    ALOGI("Jitterbuffer: mode=%d (1=SLAVE), latency=%u ms, pushed=%llu, late=%llu",
  
          mode, latency, num_pushed, num_late);
  
    
  
    gst_object_unref(session);
  
  }
  
  g_value_unset(&val);
  
  
  
  return G_SOURCE_CONTINUE;
  
  }
  
  ```
- ### 預期 Log
  
  **SLAVE mode 正常工作：**
  
  ```
  
  Jitterbuffer: mode=1 (1=SLAVE), latency=50 ms, pushed=3000, late=15
  
  Jitterbuffer: mode=1 (1=SLAVE), latency=50 ms, pushed=6000, late=28
  
  ```
- `mode=1` 確認是 SLAVE
- `late` 數字表示有些封包因為太晚而被標記（這是正常的）
  
  **沒有 SLAVE mode（問題）：**
  
  ```
  
  Jitterbuffer: mode=0, latency=100 ms, pushed=500, late=0
  
  Jitterbuffer: mode=0, latency=100 ms, pushed=1500, late=0
  
  ```
- `mode=0` 表示沒有特定模式
- `late=0` 表示從不丟棄過舊封包，導致堆積
- ## 總結
  
  | 特性 | 沒有 SLAVE | 使用 SLAVE |
  
  |------|-----------|-----------|
  
  | 延遲 | 持續增加 | 固定在 latency 值 |
  
  | Buffer 大小 | 越來越大 | 保持穩定 |
  
  | 丟包處理 | 累積問題 | 主動清理 |
  
  | 適用場景 | 檔案播放 | **實時串流** |
  
  對於你的 Miracast 應用，**SLAVE mode 是必須的**，它能確保延遲不會無限增長。