public:: true
tags:: Miracast, GStreamer

- # RTP Jitterbuffer 的 Buffer Mode
- ## 模式類型
  ```cpp
  // GStreamer 定義的模式
  enum {
    BUFFER_MODE_NONE = 0,     // 不使用特定模式
    BUFFER_MODE_SLAVE = 1,    // Slave 模式 - 跟隨 pipeline clock
    BUFFER_MODE_BUFFER = 2,   // Buffer 模式 - 自主緩衝
    BUFFER_MODE_SYNCED = 4    // Synced 模式 - RTP 時間戳同步
  };
  ```
- ### SLAVE Mode (模式 1) - 推薦用於低延遲
  
  **特性：**
- Jitterbuffer **跟隨 pipeline 的 clock**
- 根據 pipeline 當前時間決定何時輸出封包
- 如果封包太舊（相對於 pipeline clock），可以主動丟棄
- **適合實時串流**，延遲較低
  **運作方式：**
  
  ```
  Pipeline Clock: -----|-------------|-------------|----->
                   現在時間
  
             
  
  Jitterbuffer:  [舊封包] [當前] [未來封包]
  
                 ↓
  
              可能被丟棄
  
  ```
-