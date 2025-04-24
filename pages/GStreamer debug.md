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
	  
	  ```
	  // ✅ Acquire multicast lock to enable multicast packet reception
	  WifiManager wifi = (WifiManager) getApplicationContext().getSystemService(Context.WIFI_SERVICE);
	  multicastLock = wifi.createMulticastLock("gst-rtp");
	  multicastLock.setReferenceCounted(true);
	  multicastLock.acquire();
	  Log.d("Multicast", "MulticastLock acquired? " + multicastLock.isHeld())
	  
	  new Thread(() -> {
	    try {
	      MulticastSocket socket = new MulticastSocket(4002);
	      socket.setReuseAddress(true);
	      socket.joinGroup(InetAddress.getByName("224.5.5.5"));
	      byte[] buf = new byte[2048];
	      while (true) {
	        DatagramPacket packet = new DatagramPacket(buf, buf.length);
	        socket.receive(packet);
	        Log.d("MulticastTest", "Got packet: " + packet.getLength());
	      }
	    } catch (Exception e) {
	      Log.e("MulticastTest", "Error receiving multicast", e);
	    }
	  }).start();
	  ```
	- 看 udpsrc log
		- 確認 udpsrc 設定成功
		  
		  ```cpp
		  GstElement* udpsrc = gst_bin_get_by_name(GST_BIN(data->pipeline), "udpsrc0");
		    if (!udpsrc) {
		      GST_ERROR("udpsrc not found");
		    } else {
		      g_print("udpsrc element exists\n");
		      gchar* uri;
		      g_object_get(udpsrc, "uri", &uri, NULL);
		      g_print("udpsrc URI: %s\n", uri);
		      g_free(uri);
		    }
		  ```
		- 檢查 udpsrc 是否發送到下一個 element
		  
		  ```cpp
		  gst_pad_add_probe(pad, GST_PAD_PROBE_TYPE_BUFFER, [](GstPad* pad, GstPadProbeInfo* info, gpointer user_data) -> GstPadProbeReturn {
		        g_print(">>> udpsrc delivered buffer to identity\n");
		        return GST_PAD_PROBE_OK; 
		  }, NULL, NULL);
		  ```
- ### 檢查 element 有設定成功
	- ```cpp
	  GstElement* depay = gst_bin_get_by_name(GST_BIN(data->pipeline), "rtph264depay");
	    if (!depay)
	      GST_ERROR("Depayloader not found");
	  ```
- ### 檢查 appsink 設定
	- ```cpp
	  // appsinke 需要設定 emit signal 才能觸發 callback
	  g_object_set(data->app_sink, "emit-signals", TRUE, NULL);
	  g_print("Set emit-signals = TRUE on appsink\n");
	  
	  
	  // 確認 rtph264depay 有正確的輸出 h264 frame
	  // 設定 cap 的期望 format
	  // 和在 pipeline 設定一樣
	  GstCaps* sink_caps = gst_caps_new_simple("video/x-h264",
	                                           "stream-format", G_TYPE_STRING, "byte-stream",
	                                           "alignment", G_TYPE_STRING, "au",
	                                           NULL);
	  g_object_set(data->app_sink, "caps", sink_caps, NULL);
	  gst_caps_unref(sink_caps);
	  
	  // appsink 設定接到 new-sample 的 callback
	  g_signal_connect(data->app_sink, "new-sample", G_CALLBACK(new_sample_cb), data);
	  g_print("Registered appsink new-sample handler\n");
	  GstPad* sinkpad = gst_element_get_static_pad(data->app_sink, "sink");
	  gboolean is_linked = gst_pad_is_linked(sinkpad);
	  g_print("appsink is linked? %s\n", is_linked ? "yes" : "no");
	  gst_object_unref(sinkpad);
	  ```
	- 在 callback 寫 log 檢查有沒有觸發
	  
	  ```cpp
	  new_sample_cb(GstElement* sink, CustomData* data) {
	    g_print(">>> INSIDE new_sample_cb\n");
	  ```
-