- public::true
- ## 為什麼 textureRegistry_.createSurfaceTexture() 建出來的 surface 在背景回來後仍可播放？
	- 因為這建立的是 Flutter 管理的 SurfaceTexture → TextureEntry，它的生命周期不是由 Android 系統的視圖（SurfaceView）控制，而是：
	  •	Flutter engine 幫你持有這個 SurfaceTexture
	  •	即使 App 退到背景，這個 SurfaceTexture 物件仍然存在
	  •	沒有「Surface 銷毀 / 重建」事件 → 所以 GStreamer 的 sink 不會斷鏈、不會 crash
-