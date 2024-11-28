public:: true

- # RTP
  collapsed:: true
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
  collapsed:: true
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
  collapsed:: true
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
		  [Example constraint exerciser](https://developer.mozilla.org/en-US/docs/Web/API/Media_Capture_and_Streams_API/Constraints#example_constraint_exerciser)
	- ## Encoding
	  Once captured, media streams are encoded using codecs before transmission. WebRTC supports various codecs, including VP8 and VP9 for video, and Opus for audio. The choice of codec can affect the quality and bandwidth requirements of the media stream.
	  https://developer.mozilla.org/en-US/docs/Web/Media/Formats/WebRTC_codecs
	  **Bitrate Control**
	  WebRTC dynamically adjusts the encoding bitrate based on:
		- **Network Conditions:** Packet loss or bandwidth changes.
		- **Resolution and Frame Rate:** Encoding can be scaled to fit device capabilities.
	- ## Transmission
		- ### Signaling
		- ### Connection Setup
		- ### **Data Transmission**
			- WebRTC streams media data over:
				- **SRTP (Secure Real-time Transport Protocol):** Ensures secure transmission of audio and video.
				- **RTP (Real-time Transport Protocol):** For payload delivery with minimal latency.
				- **RTCP (RTP Control Protocol):** Provides feedback on quality, such as packet loss and jitter.
		- ### **Packet Prioritization**
			- Video and audio packets are prioritized differently to ensure audio continuity (e.g., in poor network conditions).
			- **Forward Error Correction (FEC):** Used to reconstruct lost packets.
	- ## Rendering
		- ### **Rendering Audio**
			- Decoded audio streams are passed to the system's audio output (e.g., speakers or headphones).
			- **Audio Rendering Components:**
				- Volume control
				- Noise suppression
				- Echo cancellation
		- ### **Rendering Video**
			- Decoded video frames are displayed in an HTML `<video>` element.
			- **Steps:**
				- The video decoder reconstructs the compressed frames.
				- Frames are queued for rendering in the browser or native application.
				- Display timing ensures synchronization with the audio stream.
- # Network Traversal
  collapsed:: true
	- ## NAT
	  NAT 是一種將內部IP 與外部IP互相轉換之技術。負責將進出封包的表頭進行轉換使得內部電腦可以 透通的與外部網路連線溝通。
		- ### Full Cone NAT
		  單純的做位址轉換，並未對進出的封包設限
		  ![image.png](../assets/image_1732760533280_0.png)
		- ### Restricted Cone NAT (Address Restricted Cone)
		  從內部送出之封包的目的地 IP 位址會被記住。只有這些曾經收過這些封包的位址可以送封包進入 NAT。由其他位址送進來的封包，都會被檔下。
		  ![image.png](../assets/image_1732760578137_0.png){:height 243, :width 604}
		- ### Port Restricted Cone NAT
		  由外部送進來的封包，除了由那些接收過內部所送出 的封包的IP 位址及 Port Number 所送來的封包之外，都會被檔下。
		  ![image.png](../assets/image_1732760613347_0.png)
		- ### Symmetric NAT
		  前三種NAT在做位址轉換時，無論封包是送往何處， NAT內部同一內部位址 都對應到同一個外部位址，但在Symmetric NAT內則每一內部位址對不同的目的地， 都對應到不同的外部位址。
		  Symmetric NAT只允許先由私有網域內的使用者發送封包到網際網路中的使用者 可以回傳封包。
		  ![image.png](../assets/image_1732761037385_0.png)
	- ## Challenges in P2P connections
	  兩個皆位於 Private IP 區域內的設備，欲建立連線時，會因為不知道對方的 Public IP Address，而無法正確的建立連線。
	- ## WebRTC tools for NAT/Firewall Traversal
		- **ICE (Interactive Connectivity Establishment):** A framework that combines STUN and TURN to discover the best path to connect peers. ICE gathers multiple connection candidates (e.g., direct, STUN-derived, TURN-relayed) and tests them to determine the most efficient route.
		- **STUN (Session Traversal Utilities for NAT):** Allows a device to discover its public IP address and the type of NAT it is behind by sending requests to a STUN server, which responds with the device's public address and port. This information helps establish direct connections when possible.
		- **TURN (Traversal Using Relays around NAT):** Used when direct connections fail, TURN servers relay data between peers. This approach is more resource-intensive but ensures connectivity in restrictive NAT scenarios, such as with symmetric NATs.
- # Security in WebRTC
	- ## DTLS
	- ## SRTP
- # Reference
	- "Introduction to WebRTC protocols," *mdn web docs*, Available: [link_to_page](https://developer.mozilla.org/en-US/docs/Web/API/WebRTC_API/Protocols).
	- "Getting started with WebRTC," *WebRTC.org*, Available: [link_to_page](https://webrtc.org/getting-started/overview).
	- "SCTP introduction,"  Available: [link_to_page](https://sites.google.com/site/applezulab/network/sctp_introduction). 
	  type:: [[Web Page]]
-