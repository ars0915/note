- MediaProjection（透過 VirtualDisplay 輸出 Surface 給 MediaCodec）會根據系統畫面更新與 VSync 決定是否送畫面更新 frame 給 Surface，所以：
  •	若螢幕沒變動，幾乎不會產生新 frame → 幀率自然變低。
  •	若應用程式或 UI 更新非常快，可能 frame rate 過高。
- 因此，你無法準確地控制每秒輸出幾幀，除非你自己控制 frame 的產生速度。
- # WebRTC 架構中的關鍵組件
	- 1. VideoSource（VideoCapturer）
		- WebRTC 的 VideoCapturer 層能以「主動推 frame」的方式控制送入幀率。
		- 與 MediaProjection 整合時，它會用 timer 控制 capture 頻率（例如 30 fps → 每 33ms 抓一次畫面）。
	- 2. VideoFrame Buffer 管線
		- 接收到的 frame 會先進入幀緩衝區，然後才進 encoder。
		- 此設計讓 WebRTC 可以丟棄 frame（FrameDropper）或根據 CPU/GPU 負載跳過畫面以維持目標 fps。
	- 3. VideoEncoder + Pacing Controller
		- VideoEncoder 並不會決定何時編碼，而是由外層的 VideoStreamEncoder 控制。
		- 此外，它會根據 leaky bucket 或 token bucket pacing 原則來節流。
	- 4. Task Queue 與 Timing Control
		- 所有的 Media I/O 都在 WebRTC 的 thread-safe TaskQueue 裡進行，並使用 high-resolution timer 控制 pacing。
		- 這保證了時間點精準度，即便 frame 來得太快也能被緩衝/丟棄/降頻處理。
- # 被動擷取模式
	- 提供一個 Surface 給 MediaProjection → 系統自動把畫面畫進來。
	  不控制「何時畫進來」，只能控制「何時送到 encoder」
	  常見例子是用 SurfaceTexture 接收 frame，再自己用 OpenGL 畫到 encoder surface
-
- ## 方案 A：主動擷取（before capture control）
  •	用 Timer 或 Handler 以固定間隔（例如每 33ms）主動從 MediaProjection 擷取畫面（透過 ImageReader.acquireLatestImage()）
  •	沒有 frame 就不擷取 → 節省 CPU/GPU
  •	掌握擷取頻率，幀率由你決定
- ##  方案 B：被動擷取 + 控制送入（after capture control）
  •	系統自動將畫面畫進 Surface（ImageReader 或 SurfaceTexture）
  •	你以固定頻率取出 frame、判斷是否丟掉或送進 MediaCodec
  •	幀率控制點在編碼前