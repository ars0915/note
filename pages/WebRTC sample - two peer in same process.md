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
			-
	-
	- ## macOS 無法連線
	-