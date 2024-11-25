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
	  **RTCRtpSender**
	  RTCRtpSender 處理 MediaStreamTrack 資料到遠端對等點的編碼和傳輸。
	  **RTCRtpReceiver**
	  RTCRtpReceivers 提供檢查和獲取有關傳入 MediaStreamTrack 資料的資訊的能力。
	  **RTCRtpTransceiver**
	  RTCRtpTransceiver 是一對共用 SDP mid 屬性的 RTP 傳送器和 RTP 接收器，這表示它們共用相同的 SDP media m-line（表示雙向 SRTP 串流）。這些由 RTCPeerConnection.getTransceivers() 方法傳回，每個 mid 和收發器共享一對一的關係，每個 RTCPeerConnection 的 mid 都是唯一的。
		- ### How to establish a RTCPeerConnection object in JavaScript.
			- The `RTCPeerConnection` is created using its constructor, which can take an optional configuration object to define connection settings.
			  ```javascript
			  const configuration = {
			    iceServers: [
			        { urls: "stun:stun.l.google.com:19302" }, // Google's public STUN server
			        // Add TURN servers here if needed
			    ],
			  };
			  
			  // Create the RTCPeerConnection object
			  const peerConnection = new RTCPeerConnection(configuration);
			  
			  console.log("RTCPeerConnection created:", peerConnection);
			  
			  // You can also create an `RTCPeerConnection` without any configuration:
			  const peerConnection = new RTCPeerConnection();
			  ```
		- ### Lifecycle
			- #### **Initialization**:
				- Create `RTCPeerConnection` objects on both sides.
				  ```javascript
				  const configuration = {
				  iceServers: [
				      { urls: "stun:stun.l.google.com:19302" }, // Google's public STUN server
				      // Add TURN servers here if needed
				  ],
				  };
				  - // Create the RTCPeerConnection object
				  const peerConnection = new RTCPeerConnection(configuration);
				  - console.log("RTCPeerConnection created:", peerConnection);
				  - // You can also create an `RTCPeerConnection` without any configuration:
				  const peerConnection = new RTCPeerConnection();
				  ```
				- Add media tracks or data channels.
			- ((67441318-7be3-4c5f-b921-e393ac007fb0))
				- **Signaling**:
					- Exchange SDP offers/answers and ICE candidates via a signaling server.
				- **ICE Gathering**:
					- Gather local candidates and send them incrementally (Trickle ICE).
				- **ICE Connectivity Checks**:
					- Test candidate pairs and select the best working path.
			- **Connection Established**:
			- Media starts flowing between peers.
			- **Renegotiation (Optional)**:
			- Triggered if media configuration changes (e.g., adding a track).
			- **Connection Termination**:
			- Close the PeerConnection when done.
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
				- 這會透過向 RTCPeerConnection 發送 `negotiationneeded` event 來觸發 RTCPeerConnection 的renegotiation，您的程式碼會回應該事件，使用 RTCPeerConnection.createOffer 產生 SDP offer，並透過 signaling server 將其傳送到 remote peer。
			- #### Remote peer
				- ```javascript
				  async function holdRequested(offer) {
				    try {
				      await peerConnection.setRemoteDescription(offer);
				      await audioTransceiver.sender.replaceTrack(null);
				      audioTransceiver.direction = "recvonly";
				      await sendAnswer();
				    } catch (err) {
				      /* handle the error */
				    }
				  }
				  ```
				  收到 `"sendonly"` 的 SDP offer 後執行
				  1. Set the remote description to the specified `offer` by calling `RTCPeerConnection.setRemoteDescription()`
				  2. Replace the audio transceiver's `RTCRtpSender`'s track with `null`, meaning no track. This stops sending audio on the transceiver.
				  3. Set the audio transceiver's `direction` property to `"recvonly"`, instructing the transceiver to only accept audio and not to send any.
				  4. The SDP answer is generated and sent using a method called `sendAnswer()`, which generates the answer using `createAnswer()` then sends the resulting SDP to the other peer over the signaling service.
		- ### 關閉 hold mode
			- #### Local
				- ```javascript
				  async function disableHold(micStream) {
				    await audioTransceiver.sender.replaceTrack(micStream.getAudioTracks()[0]);
				    audioTransceiver.receiver.track.enabled = true;
				    audioTransceiver.direction = "sendrecv";
				  }
				  ```
				  1. The audio transceiver's `RTCRtpSender`'s track is replaced with the specified stream's first audio track.
				  2. The transceiver's incoming audio track is re-enabled.
				  3. The audio transceiver's direction is set to `"sendrecv"`, indicating that it should return to both sending and receiving streamed audio, instead of only sending.
				- 這也會觸發 renegotiation
			- #### Remote
				- ```javascript
				  async function holdEnded(offer, micStream) {
				    try {
				      await peerConnection.setRemoteDescription(offer);
				      await audioTransceiver.sender.replaceTrack(micStream.getAudioTracks()[0]);
				      audioTransceiver.direction = "sendrecv";
				      await sendAnswer();
				    } catch (err) {
				      /* handle the error */
				    }
				  }
				  ```
				  收到 `"sendrecv"` 的 SDP offer 後執行
- # Connectivity
  id:: 67441318-7be3-4c5f-b921-e393ac007fb0
	- ## Signaling
		- ### Session descriptions
		  The configuration of an endpoint on a WebRTC connection.
		  透過 [[SDP]] 交換資訊
			- Example code Caller:
			  
			  ```javascript
			  const pc = new RTCPeerConnection();
			  
			  // Add local media tracks
			  const stream = await navigator.mediaDevices.getUserMedia({ audio: true, video: true });
			  stream.getTracks().forEach(track => pc.addTrack(track, stream));
			  
			  // Create an offer
			  const offer = await pc.createOffer();
			  
			  // Set the local description
			  await pc.setLocalDescription(offer);
			  
			  // Send the local description (SDP) to the remote peer via signaling
			  sendToSignalingServer(pc.localDescription);
			  
			  // Event: Handle incoming ICE candidates from the signaling server
			  signalingServer.on("iceCandidate", candidate => {
			      pc.addIceCandidate(candidate).catch(e => console.error("Error adding ICE candidate:", e));
			  });
			  
			  // Step: Receive the answer from the remote peer
			  signalingServer.on("answer", async answer => {
			      try {
			          // Set the received answer as the remote description
			          await pc.setRemoteDescription(new RTCSessionDescription(answer));
			          console.log("Remote description set successfully.");
			      } catch (e) {
			          console.error("Error setting remote description:", e);
			      }
			  });
			  ```
		- ### Pending and current descriptions
			- The **current description** (which is returned by the `RTCPeerConnection.currentLocalDescription` and `RTCPeerConnection.currentRemoteDescription` properties) represents the description currently in actual use by the connection. This is the most recent connection that both sides have fully agreed to use.
			- The **pending description** (returned by `RTCPeerConnection.pendingLocalDescription` and `RTCPeerConnection.pendingRemoteDescription`) indicates a description which is currently under consideration following a call to `setLocalDescription()` or `setRemoteDescription()`, respectively.
	- ## ICE
		- ### Trickle ICE
		  Trickle ICE is a feature in WebRTC that allows ICE candidates (potential network paths) to be sent incrementally to the remote peer as they are discovered, rather than waiting for the entire list of candidates to be gathered. This helps establish the connection faster and makes the process more efficient.
		- ### Candidate Pair
		  id:: 674449f6-4fb3-4a89-a5c9-507b51322e81
		  兩個 Peer 各自的 Candidate 組合
		- ### ICE rollbacks
		  A rollback restores the SDP offer (and the connection configuration by extension) to the configuration it had the last time the connection's `signalingState` was `stable`.
-