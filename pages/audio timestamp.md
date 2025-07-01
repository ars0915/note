## 🎵  **音頻的特性**
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
-