## 🎵  **音頻的特性**
音頻對時間戳要求嚴格，而視頻相對寬容
	- **連續性要求高** - 必須無縫播放，不能有空隙或重疊
	- **時間敏感** - 20ms 的錯位人耳就能察覺
	- **累積效應** - 小的時間戳錯誤會累積成明顯的音視頻不同步
- ## 🎥  **視頻的特性**
	- **容錯性高** - 可以跳幀、重複幀，人眼不易察覺
	- **離散性** - 幀與幀之間有自然的分界
	- **視覺緩衝** - 人眼對時間誤差的敏感度較低
- ## GStreamer 的時間
	- `GstClockTime absolute_time = gst_clock_get_time(clock);` 一直增長的絕對時間，類似系統時間
	- `GstClockTime base_time = gst_element_get_base_time(pipeline);` pipeline 進入 PLAYING 狀態時的 clock 時間戳記
	- `audio timestamp = current_time - base_time;` 時間戳反映音頻在 pipeline 中的相對位置
- ## 總是使用 pipeline 時間的問題
	- ### 網路延遲和抖動
	  ```
	  // 假設網路不穩定
	  時間點 10.0s: 收到本該在 9.8s 到達的數據包
	  時間點 10.1s: 收到本該在 9.9s 到達的數據包  
	  時間點 10.2s: 收到本該在 10.0s 到達的數據包
	  
	  // 總是用 pipeline 時間的結果：
	  timestamp = 10.0s, 10.1s, 10.2s
	  // 但實際音頻順序應該是：9.8s, 9.9s, 10.0s
	  ```
	- ### 音頻時間戳跳躍
	  ```
	  GstClockTime current_timestamp = current_time - base_time;
	  
	  // 如果 CPU 忙碌或線程調度延遲：
	  Frame 1: current_timestamp = 5.000s
	  Frame 2: current_timestamp = 5.035s  (應該是 5.020s)
	  Frame 3: current_timestamp = 5.070s  (應該是 5.040s)
	  
	  // 結果：音頻時間戳不連續，可能導致播放斷續或重複
	  ```
	- ### 與音頻數據本身的時序脫節
	  ```
	  // 音頻數據的實際時序：
	  OPUS Frame 1: 代表 0-20ms 的音頻
	  OPUS Frame 2: 代表 20-40ms 的音頻  
	  OPUS Frame 3: 代表 40-60ms 的音頻
	  
	  // 但 pipeline 時間可能是：
	  Frame 1 到達時: pipeline = 5.100s → timestamp = 5.100s
	  Frame 2 到達時: pipeline = 5.150s → timestamp = 5.150s
	  Frame 3 到達時: pipeline = 5.180s → timestamp = 5.180s
	  
	  // 音頻內容和時間戳完全不匹配！
	  ```
- ## 累積時間戳的優勢
	- 保持音頻連續性
	- 平滑播放
	- 正確的音頻時序
	  timestamp 反映音頻內容的實際時序關係，而不是網路傳輸的時序
	- **總是用 pipeline 時間** = 把網路延遲和系統抖動直接反映到音頻時間戳上，破壞了音頻的內在時序關係。
- ## timestamp 調整
	- ### timestamp 落後的可能原因
		- 網路延遲
		- 發送端中斷
		  ```
		  0s-5s: 正常發送，timestamp_ 累積到 5s
		  5s-8s: 發送端暫停（3秒靜音期）
		  8s: 重新開始發送，timestamp_ 繼續從 5s 累積
		  8s: pipeline_time = 8s, timestamp_ = 5s
		  ```
		- 接收端處理慢
		  ```
		  // 接收端 CPU 負載高，處理音頻幀很慢
		  發送端: frame1(0s) frame2(20ms) frame3(40ms) 快速發送
		  接收端: 
		    - 10:00:00.100 收到 frame1，但 10:00:00.200 才處理
		    - 10:00:00.150 收到 frame2，但 10:00:00.250 才處理
		    
		  // 處理 frame2 時：
		  // timestamp_ = 20ms（數據本身的時間）
		  // pipeline_time = 250ms（實際經過的時間）
		  // 落後 230ms！
		  ```
		- 緩衝區積壓
		-
		- **落後太多** → 數據被當作過時丟棄
	- ### timestamp 超前的可能原因
		- 累積誤差
		  ```
		  // 長時間運行後的累積誤差
		  for (int i = 0; i < 10000; i++) {
		      timestamp_ += 20 * GST_MSECOND;  // 累積 20ms
		      // 但實際間隔可能是 19.8ms 或 20.2ms
		  }
		  // 累積後 timestamp_ 可能比實際時間快幾秒
		  ```
		- 系統時間跳躍
		  pipeline_time 基於系統時鐘，如果系統時間被調整（NTP 同步、手動調整），pipeline 時鐘可能突然後退，但 timestamp_ 繼續累積
		- 處理延遲
		  ```
		  // CPU 高負載時
		  發送端: frame1(0s) frame2(20ms) frame3(40ms)
		  接收端處理: 
		    - 收到 frame1，處理延遲 → pipeline_time = 50ms, timestamp_ = 0ms
		    - 收到 frame2，快速處理 → pipeline_time = 60ms, timestamp_ = 20ms  
		    - 收到 frame3，立即處理 → pipeline_time = 61ms, timestamp_ = 40ms
		    
		  // frame3 時：timestamp_(40ms) < pipeline_time(61ms) ✓
		  // 但如果後續幀快速到達...
		    - 收到 frame4 → pipeline_time = 62ms, timestamp_ = 60ms
		    - 收到 frame5 → pipeline_time = 63ms, timestamp_ = 80ms
		    
		  // 這時 timestamp_ 開始超前了！
		  ```
		- **超前太多** → 數據被當作未來數據延遲播放
	-