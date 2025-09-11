- pipeline
  ```
  udpsrc -> rtpbin -> rtpmp2tdepay -> tsparse -> tsdemux 
  tsdemux 同時接 audio 跟 video
  tsdemux -> (audio) queue -> aacparse -> avdec_aac -> audioconvert -> audioresample -> openslessink
  tsdemux -> (video) queue -> decodebin -> videoconvert -> capsfilter -> queue -> glimagesink
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
	- 在 ts