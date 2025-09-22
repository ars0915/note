public:: true
tags:: GStreamer

- GStreamer 透過 `GstClock` object , buffer timestamp, `SEGMENT` event 去同步 pipeline 的 streams
- # Clock running-time
	- 因為電腦會有多個時間來源，所以 GStreamer 有很多 `GstClock`
	- **absolute-time** 可以透過 `gst_clock_get_time ()`取得
	- **running-time** 是由 `absolute-time - base-time` 算出
	- **base-time** 是當 GStreamer 的 pipeline 切換到 PLAYING 狀態時，選擇一個 GstClock（通常是系統時鐘或 audio/video sink 所提供的時鐘）。
	-
	-
	-
	-