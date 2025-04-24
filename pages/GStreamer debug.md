## 黑屏沒有錯誤訊息
	- 在 java 層把 gstreamer video pipeline 的內容印出來
	  `uri=udp://224.5.5.5:4002 multicast-iface=wlan0 caps="application/x-rtp, media=video, encoding-name=H264, payload=96, clock-rate=90000" ! rtph264depay ! identity name=checkbuffer silent=false ! appsink name=appsink sync=false`
	- gstreamer 除了 default 的 log 要設定 level 外，用到的 element 也要設定
	  
	  ```cpp
	  // default
	  gst_debug_set_default_threshold(GST_LEVEL_TRACE);
	  // 其他 element
	  gst_debug_set_threshold_for_name("identity", GST_LEVEL_LOG);
	  gst_debug_set_threshold_for_name("checkbuffer", GST_LEVEL_LOG);
	  gst_debug_set_threshold_for_name("udpsrc", GST_LEVEL_LOG);
	  gst_debug_set_threshold_for_name("basesrc", GST_LEVEL_LOG);
	  gst_debug_set_threshold_for_name("GST_CAPS", GST_LEVEL_LOG);
	  gst_debug_set_threshold_for_name("GST_PIPELINE", GST_LEVEL_LOG);
	  gst_debug_set_threshold_for_name("fakesink", GST_LEVEL_LOG);
	  gst_debug_set_threshold_for_name("GST_DEBUG_DUMP", GST_LEVEL_TRACE);
	  gst_debug_set_threshold_for_name("rtph264depay", GST_LEVEL_LOG);
	  ```
- ### 檢查是否 App 有接到流量
	-