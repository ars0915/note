public:: true

- 每個 RTCPeerConnection 都有方法提供對服務對等連線的 RTP 傳輸清單的存取。這些對應於 RTCPeerConnection 支援的以下三種傳輸類型：
	- **RTCRtpSender**
	  RTCRtpSender 處理 MediaStreamTrack 資料到遠端對等點的編碼和傳輸。
	- **RTCRtpReceiver**
	  RTCRtpReceivers 提供檢查和獲取有關傳入 MediaStreamTrack 資料的資訊的能力。
	- **RTCRtpTransceiver**
	  RTCRtpTransceiver 是一對共用 SDP mid 屬性的 RTP 傳送器和 RTP 接收器，這表示它們共用相同的 SDP media m-line（表示雙向 SRTP 串流）。這些由 RTCPeerConnection.getTransceivers() 方法傳回，每個 mid 和收發器共享一對一的關係，每個 RTCPeerConnection 的 mid 都是唯一的。
- ### Lifecycle
	- #### **Signaling**:
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
		- Generate offer/answer
		  
		  ```javascript
		  const offer = await pc.createOffer();
		  await pc.setLocalDescription(offer);
		  ```
		- Exchange ICE Candidates
		  Both peers gather and exchange ICE candidates incrementally (Trickle ICE) or after all candidates are collected (Full ICE).
		  ```javascript
		  pc.onicecandidate = (event) => {
		      if (event.candidate) {
		          signalingServer.send({ candidate: event.candidate });
		      }
		  };
		  ```
		- Signaling States
			- The `signalingState` property reflects the SDP exchange's progress:
				- **`stable`**: No ongoing SDP exchange.
				- **`have-local-offer`**: Local SDP offer created and set.
				- **`have-remote-offer`**: Remote SDP offer received.
				- **`closed`**: PeerConnection is closed.
			- ### **State Transition with `setRemoteDescription()`**
			  The `signalingState` in a WebRTC connection doesn't immediately transition to `have-remote-offer` when a remote SDP offer is received
			  When a remote offer is received, the application must explicitly call:
			  ```javascript
			  await pc.setRemoteDescription(description);
			  ```
			  This triggers the transition of the `signalingState`:
				- From **`stable`** to **`have-remote-offer`** if the connection is stable.
				- From other states to indicate that an offer has been processed.
	- #### **ICE Connection State Changes**:
		- The `iceConnectionState` property reflects the progress of the ICE connection:
			- **`new`**: ICE agent is gathering candidates.
			- **`checking`**: Performing connectivity checks.
			- **`connected`**: At least one working candidate pair is found.
			- **`completed`**: All connectivity checks are complete.
			- **`disconnected`**: Temporary disconnection (e.g., network issues).
			- **`failed`**: No valid candidate pair found.
			- **`closed`**: ICE connection is closed.
		- Event Listener
		  You can monitor ICE state changes:
		  ```javascript
		  pc.oniceconnectionstatechange = () => {
		      console.log("ICE connection state:", pc.iceConnectionState);
		  };
		  ```
	- #### **Media Stream Negotiation**:
	  Media stream negotiation defines how media (audio, video) is sent between peers.
		- **Step**
			- Add Media Tracks
			  Tracks are added to the PeerConnection using `addTrack()` or `addTransceiver()`.
			  ```javascript
			  stream.getTracks().forEach(track => pc.addTrack(track, stream));
			  ```
			- Offer/Answer Exchange
			  The SDP includes media information such as codecs, track identifiers, and encryption parameters.
			- Receive Remote Media
			  When the remote peer adds tracks, they are delivered through the ontrack event.
			  ```javascript
			  pc.ontrack = (event) => {
			      remoteVideoElement.srcObject = event.streams[0];
			  };
			  ```
			- Synchronize Tracks
			  Synchronization between audio and video tracks is handled by WebRTC based on the RTP timestamps.
		- **Media-Related Events**
			- `ontrack`: Triggered when a remote track is received.
			- **`onnegotiationneeded`**: Indicates a need to renegotiate the connection (e.g., when adding a new track).
			  
			  ```javascript
			  pc.onnegotiationneeded = async () => {
			      const offer = await pc.createOffer();
			      await pc.setLocalDescription(offer);
			      signalingServer.send({ sdp: pc.localDescription });
			  };
			  ```
	- #### **Connection Termination**
		- Close the PeerConnection when done.
		  
		  ```javascript
		  pc.close();
		  ```