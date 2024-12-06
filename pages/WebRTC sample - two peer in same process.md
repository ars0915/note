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
			- 從 outbound-rtp 發現沒有 encode frame
			  ```
			  I/flutter (21404): Timestamp: 1733376123810770.0, Stats Report ID: OT01V2797561059, Type: outbound-rtp
			  I/flutter (21404):   headerBytesSent: 0
			  I/flutter (21404):   transportId: T01
			  I/flutter (21404):   hugeFramesSent: 0
			  I/flutter (21404):   framesEncoded: 0
			  I/flutter (21404):   mid: 0
			  I/flutter (21404):   nackCount: 0
			  I/flutter (21404):   totalPacketSendDelay: 0.0
			  I/flutter (21404):   totalEncodeTime: 0.0
			  I/flutter (21404):   firCount: 0
			  I/flutter (21404):   pliCount: 0
			  I/flutter (21404):   packetsSent: 0
			  I/flutter (21404):   rtxSsrc: 2388816697
			  I/flutter (21404):   keyFramesEncoded: 0
			  I/flutter (21404):   retransmittedPacketsSent: 0
			  I/flutter (21404):   kind: video
			  I/flutter (21404):   targetBitrate: 277680.0
			  I/flutter (21404):   ssrc: 2797561059
			  I/flutter (21404):   active: true
			  I/flutter (21404):   qualityLimitationReason: none
			  I/flutter (21404):   qualityLimitationResolutionChanges: 0
			  I/flutter (21404):   bytesSent: 0
			  I/flutter (21404):   mediaSourceId: SV1
			  I/flutter (21404):   codecId: COT01_127_level-asymmetry-allowed=1;packetization-mode=1;profile-level-id=42e01f
			  I/flutter (21404):   totalEncodedBytesTarget: 0
			  I/flutter (21404):   framesSent: 0
			  I/flutter (21404):   qualityLimitationDurations: {other: 0.0, bandwidth: 0.0, cpu: 0.0, none: 0.027}
			  I/flutter (21404):   retransmittedBytesSent: 0
			  ```
		- ### root cause
			- SDP 會根據 peer track 調整 => **Local peer add track 後再建立 offer**
			- Remote peer 在 setRemoteDescription 後就會觸發 onTrack event => **在設定 SDP 前先 listen event**
	- ## macOS 無法連線
		- ### Debug step
			- 檢查 SDP 和 candidate 有沒有能配對的
			- 透過 peer connection event 確定 gathering 結束且
	-