public:: true

- ## textureRegistry_.createSurfaceTexture() 建出來的 surface 在背景回來後仍可播放？
	- 因為這建立的是 Flutter 管理的 SurfaceTexture → TextureEntry，它的生命周期不是由 Android 系統的視圖（SurfaceView）控制，而是：
	  •	Flutter engine 幫你持有這個 SurfaceTexture
	  •	即使 App 退到背景，這個 SurfaceTexture 物件仍然存在
	  •	沒有「Surface 銷毀 / 重建」事件 → 所以 GStreamer 的 sink 不會斷鏈、不會 crash
- ## 使用 SurfaceProducer（例如 SurfaceView / SurfaceControl）建立的 Surface 會怎樣？
	- 這些 Surface 是由 Android View 系統來控制的：
	  •	當 app 進入背景（onPause），或 SurfaceView detach，會 destroy EGL context
	  •	glimagesink / GStreamer 如果還在使用，就會報錯：