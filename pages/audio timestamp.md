## GStreamer 的時間
	- `GstClockTime absolute_time = gst_clock_get_time(clock);` 一直增長的絕對時間，類似系統時間
	- `GstClockTime base_time = gst_element_get_base_time(pipeline);` pipeline 進入 PLAYING 狀態時的 clock 時間戳記
	- `audio timestamp = current_time - base_time;` 時間戳反映音頻在 pipeline 中的相對位置
- ## 總是使用 pipeline 時間的問題
-