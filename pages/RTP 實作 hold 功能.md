public:: true

- # 開啟 hold mode
	- ## Local peer
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
	- ## Remote peer
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
- # 關閉 hold mode
	- ## Local
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
	- ## Remote
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