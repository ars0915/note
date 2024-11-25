# RTP
	- ## Offer
		- Generally low latency.
		  資料包按順序編號並帶有時間戳，以便在資料包無序到達時進行重組。這使得使用 RTP 發送的資料可以在不保證排序甚至根本不保證傳送的傳輸上傳送。
		  RTP 不僅限於在視聽通訊中使用。它可用於任何形式的連續或主動資料傳輸，包括資料流、主動徽章或狀態顯示更新，或控制和測量資訊傳輸。
	- ## Doesn't offer
		- RTP does **not** guarantee QoS.
		  它僅提供允許在堆疊中的其他位置實現 QoS 所需的資訊。
		  不處理可能需要的資源的分配或保留。
	- ## RTCPeerConnection
		- 每個 RTCPeerConnection 都有方法提供對服務對等連線的 RTP 傳輸清單的存取。這些對應於 RTCPeerConnection 支援的以下三種傳輸類型：
			- ### RTCRtpSender
			  RTCRtpSender 處理 MediaStreamTrack 資料到遠端對等點的編碼和傳輸。
			- ### RTCRtpReceiver
			  RTCRtpReceivers 提供檢查和獲取有關傳入 MediaStreamTrack 資料的資訊的能力。連線的接收者可以透過呼叫 RTCPeerConnection.getReceivers() 來取得。
			- ### RTCRtpTransceiver
			  RTCRtpTransceiver 是一對共用 SDP mid 屬性的 RTP 傳送器和 RTP 接收器，這表示它們共用相同的 SDP 媒體 m-line（表示雙向 SRTP 串流）。這些由 RTCPeerConnection.getTransceivers() 方法傳回，每個 mid 和收發器共享一對一的關係，每個 RTCPeerConnection 的 mid 都是唯一的。
		-
-