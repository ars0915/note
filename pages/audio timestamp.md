## GStreamer 的時間
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