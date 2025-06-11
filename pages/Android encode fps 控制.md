- MediaProjection（透過 VirtualDisplay 輸出 Surface 給 MediaCodec）會根據系統畫面更新與 VSync 決定是否送畫面更新 frame 給 Surface，所以：
  •	若螢幕沒變動，幾乎不會產生新 frame → 幀率自然變低。
  •	若應用程式或 UI 更新非常快，可能 frame rate 過高。
- 因此，你無法準確地控制每秒輸出幾幀，除非你自己控制 frame 的產生速度。
- WebRTC