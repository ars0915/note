public:: true
tags:: Multicast, GStreamer

- pipeline
  ```
  udpsrc -> rtpbin -> rtpmp2tdepay -> tsparse -> tsdemux 
  tsdemux
  |_> (audio) queue -> aacparse -> avdec_aac -> audioconvert -> audioresample -> openslessink
  |_> (video) queue -> decodebin -> videoconvert -> capsfilter -> queue -> glimagesink
  ```
- decodebin -> videoconvert -> capsfilter -> glimagesink 黑畫面
	- 看 native windows 尺寸是 1*1 => 先在 Surface 設定尺寸後再傳入
- render 完第一個畫面後就卡住
	- decodebin 也沒有再產出，應該是 glimagesink 的背壓造成
	- 前面加上 queue => decodebin -> videoconvert -> capsfilter -> queue -> glimagesink
- glimagesink 參數
	- sync = FALSE：
		- glimagesink 不根據 PTS 與 pipeline 的 clock 播放，只要有 buffer 就立刻顯示。
		  id:: 68c22b60-7cf6-4adb-84ce-69ddeafc2023
		  •	畫面會比較流暢，但幀不一定準時（甚至跑很快）。
		  •	音訊和視訊失去同步，尤其當 openslessink（音訊）是 sync = TRUE 時，就會導致：
		  •	音訊播放速度不正常（常是太快）
		  •	音訊 buffer 播放完但沒有新的到來 → 播放中止（無聲音）
	- sync = TRUE（預設）：
		- glimagesink 會根據 buffer 的 PTS（presentation timestamp） 和 pipeline 的 全域 clock（通常由音訊 sink 控制） 來同步播放。
		  •	必須等到時間到了才會顯示 → 符合 A/V sync 要求
		  •	若 render（例如 GPU 或 Surface）太慢，或是上游 PTS 很跳躍，就會：
		  •	畫面掉幀（因為 PTS 過期）
		  •	視訊嚴重卡頓，但音訊正常
- decodebin 接 probe 看輸出發現一秒一幀
	- 在 tsdemux 發現每 33ms 就有一幀、也有 PTS
	- 使用 tsdemux -> queue -> h264parse -> capsfilter -> filesink 產出 h264 檔案可以播放
	- 加了 probe 發現 tsdemux 和 h264parse PTS 都是幾乎 33ms 差距
	- 改用 avdec_h264: queue ! h264parse ! avdec_h264 ! videoconvert ! render_queue ! glimagesink 只有一幀就卡住了
	- 查看 element 輸出 PTS
	  ```cpp
	  gst_pad_add_probe(
	        gst_element_get_static_pad(self->video_sink_, "sink"), // element, sink/src
	        GST_PAD_PROBE_TYPE_BUFFER,
	        [](GstPad* pad, GstPadProbeInfo* info, gpointer) -> GstPadProbeReturn {
	          GstBuffer* buffer = GST_PAD_PROBE_INFO_BUFFER(info);
	          GstClockTime pts = GST_BUFFER_PTS(buffer);
	          g_print("[%s] : PTS = %" GST_TIME_FORMAT "\n", GST_ELEMENT_NAME(GST_PAD_PARENT(pad)), GST_TIME_ARGS(pts));
	          static GstClockTime last_time = GST_CLOCK_TIME_NONE;
	          GstClockTime now = gst_util_get_timestamp();
	          if (last_time != GST_CLOCK_TIME_NONE) {
	            GstClockTime diff = now - last_time;
	            double sec = GST_TIME_AS_SECONDS(diff);
	            if (sec > 0.01) {  // 避免太短一堆 0.000
	              g_print("[%s]  interval: %" GST_TIME_FORMAT "\n", GST_ELEMENT_NAME(GST_PAD_PARENT(pad)), GST_TIME_ARGS(diff));
	            }
	          }
	          last_time = now;
	          return GST_PAD_PROBE_OK;
	        },
	        nullptr, nullptr);
	  ```
- decodebin PTS 33ms
	- 查看 decodebin 選用什麼解碼器
	  ```cpp
	  // 在建立 decodebin 時連接 signal
	  g_signal_connect(decodebin, "element-added", G_CALLBACK(on_element_added), NULL);
	  
	  static void on_element_added(GstBin* bin, GstElement* element, gpointer user_data) {
	      const gchar* element_name = gst_element_get_name(element);
	      const gchar* factory_name = gst_plugin_feature_get_name(
	          GST_PLUGIN_FEATURE(gst_element_get_factory(element)));
	      
	      ALOGI("decodebin added element: %s (factory: %s)", element_name, factory_name);
	      
	      // 特別留意解碼器
	      if (strstr(factory_name, "dec") || strstr(factory_name, "decoder")) {
	          ALOGI("*** DECODER SELECTED: %s ***", factory_name);
	      }
	  }
	  ```
	  decodebin added element: h264parse0 (factory: h264parse)
	  decodebin added element: capsfilter0 (factory: capsfilter)
	  decodebin added element: avdec_h264-0 (factory: avdec_h264)
	- 輸出到檔案看看
	  ```
	  self->video_sink_ = gst_element_factory_make("filesink", "video_sink");
	      if (self->video_sink_) {
	        g_object_set(self->video_sink_,
	                     "location", "/sdcard/test_output.yuv",
	                     NULL);
	  ```
- queue 的 "leaky" 屬性
	- 當 queue 滿了時，要「丟棄」哪邊的 buffer：0 = 不丟（預設）1 = upstream（丟舊的 input）2 = downstream（丟新的 output）
	- upstream: **適合低延遲場景**：視訊監控、即時通訊
	- downstream: 不建議用在 video，會增加延遲
- openslessink 參數
	- buffer-time
		- ✅ 定義：
		  •	sink 內部的總 buffer 大小（以時間表示）。
		  •	GStreamer 播放音訊前，最多會累積這麼多資料進入 sink。
		- 📌 影響：
		  •	越大 → 播放越穩定（耐 jitter），但延遲變高。
		  •	越小 → 播放延遲變低，但更容易出現「聲音斷裂」、「resync」。
	- latency-time
		- ✅ 定義：
		  •	從 buffer 開始接收資料到實際播放的延遲時間。
		  •	是這個 sink 對 pipeline 提出的最小 latency 要求。
		- 📌 影響：
		  •	GStreamer 播放器會根據它與其他元素（如 demuxer）計算 latency。
		  •	如果低於 source 實際提供資料的時間，就會造成掉幀、跳音。
	- 實測結果 當 pipeline clock provider 是 audio sink 時
	  buffer-time: 200ms, latency-time: 20ms => 聲音正常，video很卡
	  buffer-time: 300ms, latency-time: 200ms => video 很順暢，audio 會一直 resync 沒聲音
-