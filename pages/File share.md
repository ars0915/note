public:: true

- ## Bytello
	- IFP <-> Device 互傳檔案
	  檔案限制 1GB
	  可以接受或拒絕發送請求
	  沒有加密
	  
	  device -> IFP  	在 IFP 同意後向 IFP 發起 TCP 連線（和原本連線的 port 不同 	使用 HTTP 發送檔案 POST /api/localshare/v1/upload HTTP/1.1  	多個 tcp segments
	  
	  IFP -> device
	  	在 device 同意後向 IFP 發起 TCP 連線（和原本連線的 port 不同
	  	可同時傳給多個 device
	  
	  發送端送出多個 Seq 後(Seq = last Seq + len, ACK = 1）
	  接收端回應 Seq = 1, ACK = last Seq + len
	  
	  Android 版不能上傳檔案
	  無法 Device <-> Device
- ## Note
	- ### 實作需注意
		- 切片
		  還原切片、拼接
		  中斷後續傳
		- 討論
			- 是否需要雲端儲存 	存取時限
			  裝置間互傳是否直接建立連線
			  是否需加密
			  一次傳的檔案數量上限
			  檔案大小上限
- ## 方案
	- webRTC
	- WebTorrent
	- ![image.png](../assets/image_1743577154099_0.png)
- ## 參考產品
	- LocalSend
		- 開源
		  使用 TCP, UDP
		  加密
		  搜尋設備使用 UDP 廣播 or HTTP 向本地所有 IP 發送請求註冊
		  傳輸使用 HTTP，接收端為 server
		  接收方無法使用時可以由發送端提供未加密 HTTP 網址下載
	- Landrop
		- 舊版開源
		  搜尋設備使用 UDP 廣播
		  每個裝置啟一個 server 使用 TCP 連線
	- AirDroid
	- drop.lol
		- 開源
		  文件說使用 webRTC
		  加密
	- ShareDrop
		- 開源
		  文件說使用 webRTC
	- WebTorrent
		- base on webRTC
		  上傳者只要傳部份給所有人
		  大家 share chunk
		  可以並行傳輸增加速度和量級
		  減少上傳者的 loading
		- 需要其中一個
			- Trackers (via WebSocket) to find other peers (用來建立 WebRTC 連線)
			- Distributed Hash Tables (DHT) in native environments (完全去中心化，不需 Signaling server)
		- 不需要所有 peer 都連線
		- 隨時可加入
		- flutter 實作方法
			- 切檔案
			- 建立Chunk Map (Bitfield)
			  `[true, false, true, false, false, true, ...]`
			- Exchange Bitfields Between Peers
			- Send/Respond to Chunk Requests
			- 選擇 Peer 的策略
				- round-robin / least-busy
		-
- ## 評估項目
	- internet / intranet 支援
	- 使用者間要透過 IFP 中轉或直連
	- housekeeping
- ## Sender 間建立連線
  避免 receiver 成為瓶頸，同時保持整個系統的高效率和彈性。
	- 在 sender 間建立直接的 WebRTC 連接
	- Mesh 網絡：每個 sender 維持與其他幾個 sender 的連接，形成分散式網絡
	- receiver 可以作為 signaling server，幫助 sender 間建立連接，但實際資料傳輸不經過 receiver
	- ### 建立連線需要的資源和時間
		- WebRTC 連線建立時間
		- 信令交換延遲
		- 初始連線建立後的檔案分割與傳輸協調
	- ### 替代解決方案
	  如果檔案通常很大（幾百 MB 或更大），則連線建立的時間相對於總傳輸時間來說就不那麼重要了，按需建立連線是合理的。但對於頻繁分享小檔案的場景，可能需要考慮更積極的連線預建策略。
		- 當有新的 sender 加入系統時，就預先與其他 sender 建立連線
			- 優點：需要傳檔時可以立即開始，無需等待連線建立
			- 缺點：消耗資源維護可能不會用到的連線
		- 維持一個基本的最小連線網路拓撲（目前 sender 都和 receiver 建立了連線就是一種星狀拓撲）
			- 只在需要額外頻寬時才動態增加更多連線
		- 連線池
			- 預先建立少量常用連線，並保持活躍
			- 使用 LRU (Least Recently Used) 策略管理連線池大小
			- 當需要新連線時，優先使用池中已有連線
		- 漸進式檔案傳輸
			- 先用已有的連線（如與 receiver 的連線）開始檔案傳輸
			- 同時在背景建立 sender 間的連線
			- 一旦 sender 間連線建立，自動轉移部分傳輸負載到新連線
		-
	-
-
-