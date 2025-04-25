public:: true

- ## 黑屏沒有錯誤訊息
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
	  // appsink 需要設定 emit signal 才能觸發 callback
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
	  
	  // 檢查 appsink 跟上一層有接好
	  GstPad* sinkpad = gst_element_get_static_pad(data->app_sink, "sink");
	  gboolean is_linked = gst_pad_is_linked(sinkpad);
	  g_print("appsink is linked? %s\n", is_linked ? "yes" : "no");
	  gst_object_unref(sinkpad);
	  ```
	- 在 callback 寫 log 檢查有沒有觸發
	  
	  ```cpp
	  new_sample_cb(GstElement* sink, CustomData* data) {
	    g_print(">>> INSIDE new_sample_cb\n");
	    
	    g_signal_emit_by_name(sink, "pull-sample", &sample);
	    if (sample) {
	      GST_INFO("pull sample");
	  
	      GstCaps* caps = gst_sample_get_caps(sample);
	      gchar* caps_str = gst_caps_to_string(caps);
	      g_print("appsink sample caps: %s\n", caps_str);
	      g_free(caps_str);
	  ```
- ## app sink callback 成功觸發
	- 在 callback function 中會 call `g_rtp_player->OnVideoFrame`，檢查 decode 有沒有正確設定
	  如果是在 cpp 設定 decode 會用到 `AMediaCodec`
	  確認 codec 和使用 hardware 或 software
	  
	  ```cpp
	  VideoDecoderPtr CreateVideoDecoder(
	    ...){
	    ...
	    ALOGI("🎬 [CreateVideoDecoder] codec type: %d", codec_type);
	    ALOGI("🎬 [CreateVideoDecoder] using hardware: %d", !use_software_decoder);
	  }
	  ```
	- 發現 hardware decode error log
	  
	  ```
	  E MediaCodec: Invalid to call at Released state; only valid in executing state
	  ```
	- #### 確認 appsink 有沒有接到 buffer
	  ```cpp
	  gst_pad_add_probe(appsink_pad, GST_PAD_PROBE_TYPE_BUFFER,
	    [](GstPad*, GstPadProbeInfo*, gpointer) -> GstPadProbeReturn {
	        g_print("🎯 Buffer arrived at appsink!\n");
	        return GST_PAD_PROBE_OK;
	    }, NULL, NULL);
	  ```
	  發現有收到 buffer
	  new-sample callback 有觸發了，但 decoder 有 error
	  ```
	  04-24 11:21:54.663 17562 17776 I GLib+stdout: PTS: 0:10:26.755039437
	  04-24 11:21:54.663 17562 17776 D MirrorPlugin: MediaSession::OnVideoFrame
	  04-24 11:21:54.663 17562 17776 D MirrorPlugin: After HandleVideoCsd
	  04-24 11:21:54.663 17562 17776 D MirrorPlugin: no video_decoder_
	  ```
	  重啟後又突然好了
- ## app sink callback 沒有觸發
	- 看上一個 element 有沒有正常運作
	  使用 `gst_bin_get_by_name` 需要在 pipeline 設定名稱，不然會是預設的像是 rtph264depay0
	- 觀察 rtph264depay 的 log
	  
	  ```apl
	  D GStreamer+GST_PADS: 0:00:09.441286955 0xb4000072899f1700 ../gst/gstpad.c:4373:gst_pad_peer_query:<rtph264depay0:src> query failed
	  D GStreamer+rtph264depay: 0:00:09.441296339 0xb4000072899f1700 ../gst/rtp/gstrtph264depay.c:395:gst_rtp_h264_depay_set_output_caps:<rtph264depay0> downstream ALLOCATION query failed
	  V GStreamer+GST_REFCOUNTING: 0:00:09.441310416 0xb4000072899f1700 ../gst/gstminiobject.c:661:gst_mini_object_unref 0xb40000726913ab70 unref 1->0
	  V GStreamer+GST_REFCOUNTING: 0:00:09.441320301 0xb4000072899f1700 ../gst/gstminiobject.c:661:gst_mini_object_unref 0xb40000726913a990 unref 7->6
	  V GStreamer+structure: 0:00:09.441328839 0xb4000072899f1700 ../gst/gststructure.c:547:gst_structure_free free structure 0xb40000725e05aac0
	  V GStreamer+GST_REFCOUNTING: 0:00:09.441337339 0xb4000072899f1700 ../gst/gstminiobject.c:661:gst_mini_object_unref 0xb40000726913a990 unref 6->5
	  V GStreamer+GST_REFCOUNTING: 0:00:09.441347801 0xb4000072899f1700 ../gst/gstminiobject.c:479:gst_mini_object_ref 0xb400007289bfd230 ref 2->3
	  V GStreamer+GST_REFCOUNTING: 0:00:09.441355763 0xb4000072899f1700 ../gst/gstminiobject.c:661:gst_mini_object_unref 0xb400007289c21350 unref 5->4
	  V GStreamer+GST_REFCOUNTING: 0:00:09.441363070 0xb4000072899f1700 ../gst/gstobject.c:265:gst_object_unref:<rtph264depay0> 0xb40000725b0ea2a0 unref 2->1
	  D GStreamer+GST_PADS: 0:00:09.441373032 0xb4000072899f1700 ../gst/gstpad.c:5954:gst_pad_send_event_unchecked:<rtph264depay0:sink> sent event, ret ok
	  V GStreamer+GST_REFCOUNTING: 0:00:09.441382724 0xb4000072899f1700 ../gst/gstminiobject.c:479:gst_mini_object_ref 0xb400007289c21350 ref 4->5
	  V GStreamer+GST_PADS: 0:00:09.441391532 0xb4000072899f1700 ../gst/gstpad.c:5387:store_sticky_event:<rtph264depay0:sink> stored sticky event caps
	  D GStreamer+GST_PADS: 0:00:09.441445301 0xb4000072899f1700 ../gst/gstpad.c:5393:store_sticky_event:<rtph264depay0:sink> notify caps
	  V GStreamer+GST_REFCOUNTING: 0:00:09.441461955 0xb4000072899f1700 ../gst/gstobject.c:238:gst_object_ref:<rtph264depay0> 0xb40000725b0ea2a0 ref 1->2
	  V GStreamer+GST_PROPERTIES: 0:00:09.441474570 0xb4000072899f1700 ../gst/gstobject.c:473:gst_object_dispatch_properties_changed:<rtph264depay0> deep notification from sink (caps)
	  V GStreamer+GST_REFCOUNTING: 0:00:09.441486955 0xb4000072899f1700 ../gst/gstobject.c:238:gst_object_ref:<pipeline0> 0xb40000725b0fa1d0 ref 1->2
	  V GStreamer+GST_REFCOUNTING: 0:00:09.441497301 0xb4000072899f1700 ../gst/gstobject.c:265:gst_object_unref:<rtph264depay0> 0xb40000725b0ea2a0 unref 2->1
	  V GStreamer+GST_PROPERTIES: 0:00:09.441505916 0xb4000072899f1700 ../gst/gstobject.c:473:gst_object_dispatch_properties_changed:<pipeline0> deep notification from sink (caps)
	  V GStreamer+GST_REFCOUNTING: 0:00:09.441515224 0xb4000072899f1700 ../gst/gstobject.c:265:gst_object_unref:<pipeline0> 0xb40000725b0fa1d0 unref 2->1
	  V GStreamer+GST_REFCOUNTING: 0:00:09.441524839 0xb4000072899f1700 ../gst/gstminiobject.c:661:gst_mini_object_unref 0xb400007289c21350 unref 5->4
	  V GStreamer+GST_PADS: 0:00:09.441534378 0xb4000072899f1700 ../gst/gstpad.c:5578:gst_pad_push_event_unchecked:<udpsrc0:src> sent event 0xb400007289c21350 (caps) to peerpad <rtph264depay0:sink>, ret ok
	  V GStreamer+GST_REFCOUNTING: 0:00:09.441549339 0xb4000072899f1700 ../gst/gstobject.c:265:gst_object_unref:<rtph264depay0:sink> 0xb40000725b0e03e0 unref 2->1
	  ```
		- `<udpsrc0:src> sent event (caps) to peerpad <rtph264depay0:sink>, ret ok`
		  代表 udpsrc0 有 push caps 且 rtph264depay0 接受
		- ```
		  <rtph264depay0:src> query failed
		  gst_rtp_h264_depay_set_output_caps:<rtph264depay0> downstream ALLOCATION query failed
		  ```
		  代表 rtph264depay 有試著 push H264 frame，但失敗了
		- **ALLOCATION query failed** 的可能原因
			- rtph264depay is trying to output something like `video/x-h264`, `stream-format=(string)byte-stream`, `alignment=(string)au`
			  If your appsink has different caps or is too strict, it will reject the caps and the chain breaks.
			  => appsink 沒有設定正確的 caps
		- **確認 rtph264depay 輸出的格式**
		  
		  ```
		  GStreamer+basetransform: 0:10:44.725148578 0xb40000725e813300 ../libs/gst/base/gstbasetransform.c:475:gst_base_transform_transform_caps:<checkbuffer>   to: video/x-h264, stream-format=(string)avc, alignment=(string)au, codec_data=(buffer)01420032ffe1001427420032898a3805e817bf34d40404041e1108cf01000428ce3c80, level=(string)5, profile=(string)baseline
		  ```
		  發現 stream-format= avc，跟 appsink 設定的 caps 不同，會被濾掉
			- #### 當 rtph264depay outputs multiple formats，appsink 需要加上 caps filter，
			  id:: 680b020b-bbe4-463c-8ec5-06cb90431d0e
				- 不然會不知道要怎麼處理收到的 data
				  ```
				  video/x-h264, stream-format=(string)avc, alignment=(string)au;
				  video/x-h264, stream-format=(string)byte-stream, alignment=(string){nal, au}
				  ```
				  	•	If you don’t set specific caps on appsink, and its caps are left as ANY, the caps negotiation might succeed, fail silently, or settle on a format the app doesn’t understand — leading to:
				  	•	**new-sample callback not called**.
				  	•	Or unexpected buffer formats.
				  	•	Or buffers arriving but decoder (if any) fails — resulting in black screen.
-