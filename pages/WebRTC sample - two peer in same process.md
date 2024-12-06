# Purpose
	- Create two peer connection in a process, connect two peer through WebRTC.
	  like https://webrtc.github.io/samples/src/content/peerconnection/pc1/
- # Initialize
	- Implement by flutter_webrtc, test on macOS and Android.
- # Final flow
	-
- # Issues
	- ## Android 連線後 remote render 失敗
	  從 RTCIceConnectionState.RTCIceConnectionStateConnected 得知連線沒有問題
		- ### Debug step
			- 發現 remote peer 沒有接到 track
			  
			  ```dart
			  remotePeer!.onTrack = (RTCTrackEvent event) {
			    if (event.streams.isNotEmpty) {
			      remoteRenderer!.srcObject = event.streams[0];
			    }
			  };
			  ```
			- 確認 ICE 有成功的 candidate pair
			  ```
			  I/flutter (12055): Timestamp: 1733369125731515.0, Stats Report ID: CP5AZpbYeE_mYhFPvWx, Type: candidate-pair
			  I/flutter (12055):   transportId: T01
			  I/flutter (12055):   requestsSent: 1
			  I/flutter (12055):   localCandidateId: I5AZpbYeE
			  I/flutter (12055):   bytesSent: 0
			  I/flutter (12055):   bytesDiscardedOnSend: 0
			  I/flutter (12055):   priority: 9114756780654345726
			  I/flutter (12055):   requestsReceived: 0
			  I/flutter (12055):   writable: true
			  I/flutter (12055):   remoteCandidateId: ImYhFPvWx
			  I/flutter (12055):   bytesReceived: 0
			  I/flutter (12055):   packetsReceived: 0
			  I/flutter (12055):   responsesSent: 0
			  I/flutter (12055):   packetsDiscardedOnSend: 0
			  I/flutter (12055):   nominated: false
			  I/flutter (12055):   packetsSent: 0
			  I/flutter (12055):   totalRoundTripTime: 0.001
			  I/flutter (12055):   responsesReceived: 1
			  I/flutter (12055):   state: succeeded
			  I/flutter (12055):   currentRoundTripTime: 0.001
			  I/flutter (12055):   consentRequestsSent: 0
			  ```
				- `bytesSent: 0` and `bytesReceived: 0`
			- 檢查 SDP
			  offer SDP
			  ```
			  a=recvonly
			  ```
			  需要有 `a=sendrecv` 或 `a=sendonly`
				- **Inspect Offer SDP After Track Addition**: 確認 add track 再建立 offer
			- 從 outbound
	-
	- ## macOS 無法連線
	-