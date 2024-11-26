public:: true

- # RTP
	- ## Offer
		- Generally low latency.
		- 資料包按順序編號並帶有時間戳，以便在資料包無序到達時進行重組。這使得使用 RTP 發送的資料可以在不保證排序甚至根本不保證傳送的傳輸上傳送。
		- RTP 不僅限於在視聽通訊中使用。它可用於任何形式的連續或主動資料傳輸，包括資料流、主動徽章或狀態顯示更新，或控制和測量資訊傳輸。
	- ## Doesn't offer
		- RTP does **not** guarantee QoS.
		  它僅提供允許在堆疊中的其他位置實現 QoS 所需的資訊。
		- 不處理可能需要的資源的分配或保留。
	- ## [[RTCPeerConnection]]
	- ## [[RTP 實作 hold 功能]]
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
	- ## [[Perfect negotiation pattern]]
	-
- # Data channel
  collapsed:: true
	- ## Create a data channel
		- ### Automatic negotiation
		  ```javascript
		  let dataChannel = pc.createDataChannel("MyApp Channel");
		  
		  dataChannel.addEventListener("open", (event) => {
		    beginTransmission(dataChannel);
		  });
		  ```
		  Call `createDataChannel()`without specifying a value for the `negotiated` property, or specifying the property with a value of `false`. This will automatically trigger the `RTCPeerConnection` to handle the negotiations for you, causing the remote peer to create a data channel and linking the two together across the network.
		- ### Manual negotiation
		  ```javascript
		  let dataChannel = pc.createDataChannel("MyApp Channel", {
		  negotiated: true,
		  });
		  - dataChannel.addEventListener("open", (event) => {
		  beginTransmission(dataChannel);
		  });
		  - requestRemoteChannel(dataChannel.id);
		  ```
		  Specifying in the options a `negotiated` property set to `true`. This signals to the peer connection to not attempt to negotiate the channel on your behalf.
	- ## Security
		- All data transferred using WebRTC is encrypted. In the case of `RTCDataChannel`, the encryption used is Datagram Transport Layer Security (DTLS)
- # [[Observe WebRTC Signaling Using Chrome Tools]]
- # Media Flow
	- ## Media capture and constraints
	  Media capture in WebRTC is facilitated by the `getUserMedia()` method, which prompts the user for permission to access their camera and microphone. Upon consent, it returns a `MediaStream` object containing `MediaStreamTrack` objects for each media type (audio and video)
	  ```javascript
	  const openMediaDevices = async (constraints) => {
	      return await navigator.mediaDevices.getUserMedia(constraints);
	  }
	  
	  try {
	      const stream = openMediaDevices({'video':true,'audio':true});
	      console.log('Got MediaStream:', stream);
	  } catch(error) {
	      console.error('Error accessing media devices.', error);
	  }
	  ```
		- ### MediaStream
		  Represents a stream of media content, which may include multiple tracks such as audio and video.
		- ### MediaStreamTrack
		  Represents a single media track within a `MediaStream`, such as an individual audio or video track.
		  Each `MediaStreamTrack` may have one or more channels.
		  The channel represents the smallest unit of a media stream, such as an audio signal associated with a given speaker, like left or right in a stereo audio track.
		- ### Constraints
		  Constraints are used to specify desired properties for media capture, such as resolution for video or sample rate for audio.
	- ## Encoding
	- ## Transmission
	- ## Rendering
	-
-
-
- # Reference
	- "Introduction to WebRTC protocols," *mdn web docs*, Available: [link_to_page](https://developer.mozilla.org/en-US/docs/Web/API/WebRTC_API/Protocols).
	- "Getting started with WebRTC," *WebRTC.org*, Available: [link_to_page](https://webrtc.org/getting-started/overview).