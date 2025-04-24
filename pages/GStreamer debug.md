## 黑屏沒有錯誤訊息
	- 在 java 層把 gstreamer video pipeline 的內容印出來
	  `udpsrc uri=udp://224.5.5.5:4002 multicast-iface=wlan0 caps="application/x-rtp, media=video, payload=96, clock-rate=90000, encoding-name=H264" ! rtpbin ! queue ! rtph264depay ! video/x-h264, stream-format=byte-stream , alignment=au ! appsink name=appsink sync=false`
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
- ### 確認 rtp 有正確送出
	- 透過 VLC 開 SDP 播放
	  
	  ```sdp
	  v=0
	  o=- 0 0 IN IP4 127.0.0.1
	  s=Pion WebRTC
	  c=IN IP4 224.5.5.5
	  t=0 0
	  m=audio 4000 RTP/AVP 111
	  a=rtpmap:111 OPUS/48000/2
	  m=video 4002 RTP/AVP 96
	  a=rtpmap:96 H264/90000
	  ```
- ### 檢查是否 App 有接到流量
	- 抓流量的 App 不一定準確
	- 檢查 JAVA 層是否有接到封包
		-