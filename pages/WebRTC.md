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
	  每個 RTCPeerConnection 都有方法提供對服務對等連線的 RTP 傳輸清單的存取。這些對應於 RTCPeerConnection 支援的以下三種傳輸類型：
		- ### RTCRtpSender
		  RTCRtpSender 處理 MediaStreamTrack 資料到遠端對等點的編碼和傳輸。
		- ### RTCRtpReceiver
		  RTCRtpReceivers 提供檢查和獲取有關傳入 MediaStreamTrack 資料的資訊的能力。
		- ### RTCRtpTransceiver
		  RTCRtpTransceiver 是一對共用 SDP mid 屬性的 RTP 傳送器和 RTP 接收器，這表示它們共用相同的 SDP media m-line（表示雙向 SRTP 串流）。這些由 RTCPeerConnection.getTransceivers() 方法傳回，每個 mid 和收發器共享一對一的關係，每個 RTCPeerConnection 的 mid 都是唯一的。
	- ## RTP 實作 hold 功能
		- ### 開啟 hold mode
			- #### Local peer
				- ```javascript
				  async function enableHold(audioStream) {
				    try {
				      await audioTransceiver.sender.replaceTrack(audioStream.getAudioTracks()[0]);
				      audioTransceiver.receiver.track.enabled = false;
				      audioTransceiver.direction = "sendonly";
				    } catch (err) {
				      /* handle the error */
				    }
				  }
				  ```
				  1. Replace their outgoing audio track with a `MediaStreamTrack` containing hold music.
				  2. Disable the incoming audio track.
				  3. Switch the audio transceiver into send-only mode.
				- 這會透過向 RTCPeerConnection 發送協商所需事件來觸發 RTCPeerConnection 的重新協商，您的程式碼會回應該事件，使用 RTCPeerConnection.createOffer 產生 SDP 報價，並透過信令伺服器將其傳送到遠端對等點。
				  包含要播放的音訊而不是本地對等點的麥克風音訊的音訊串流可以來自任何地方。一種可能性是擁有隱藏的 <audio> 元素並使用 HTMLAudioElement.captureStream() 來取得其音訊串流。
			- #### Remote peer
			-
		- ### 關閉 hold mode