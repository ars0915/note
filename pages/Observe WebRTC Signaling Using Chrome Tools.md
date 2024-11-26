# WebSocket Message
	- ## Local SDP offer
	  ![image.png](../assets/image_1732606209368_0.png)
	- ## Remote SDP answer
	  ![image.png](../assets/image_1732606328076_0.png)
	  ```
	  v=0
	  o=- 5513141044946050259 2 IN IP4 127.0.0.1
	  s=-
	  t=0 0
	  a=group:BUNDLE 0 1 2
	  a=extmap-allow-mixed
	  a=msid-semantic: WMS
	  m=audio 9 UDP/TLS/RTP/SAVPF 111
	  c=IN IP4 0.0.0.0
	  a=rtcp:9 IN IP4 0.0.0.0
	  a=ice-ufrag:D9lK
	  a=ice-pwd:Gq8blCpAHeOk23GNchayvj4o
	  a=ice-options:trickle renomination
	  a=fingerprint:sha-256 9E:65:BD:07:A7:F6:DD:05:A9:0F:23:88:8C:8F:AF:6A:64:30:C7:A3:6D:AC:2E:E5:16:D9:08:0B:55:A0:31:3F
	  a=setup:active
	  a=mid:0
	  a=extmap:1 urn:ietf:params:rtp-hdrext:ssrc-audio-level
	  a=extmap:2 http://www.webrtc.org/experiments/rtp-hdrext/abs-send-time
	  a=extmap:3 http://www.ietf.org/id/draft-holmer-rmcat-transport-wide-cc-extensions-01
	  a=extmap:4 urn:ietf:params:rtp-hdrext:sdes:mid
	  a=recvonly
	  a=rtcp-mux
	  a=rtpmap:111 opus/48000/2
	  a=rtcp-fb:111 transport-cc
	  a=fmtp:111 minptime=10;useinbandfec=1
	  m=video 9 UDP/TLS/RTP/SAVPF 106
	  c=IN IP4 0.0.0.0
	  a=rtcp:9 IN IP4 0.0.0.0
	  a=ice-ufrag:D9lK
	  a=ice-pwd:Gq8blCpAHeOk23GNchayvj4o
	  a=ice-options:trickle renomination
	  a=fingerprint:sha-256 9E:65:BD:07:A7:F6:DD:05:A9:0F:23:88:8C:8F:AF:6A:64:30:C7:A3:6D:AC:2E:E5:16:D9:08:0B:55:A0:31:3F
	  a=setup:active
	  a=mid:1
	  a=extmap:14 urn:ietf:params:rtp-hdrext:toffset
	  a=extmap:2 http://www.webrtc.org/experiments/rtp-hdrext/abs-send-time
	  a=extmap:13 urn:3gpp:video-orientation
	  a=extmap:3 http://www.ietf.org/id/draft-holmer-rmcat-transport-wide-cc-extensions-01
	  a=extmap:5 http://www.webrtc.org/experiments/rtp-hdrext/playout-delay
	  a=extmap:6 http://www.webrtc.org/experiments/rtp-hdrext/video-content-type
	  a=extmap:7 http://www.webrtc.org/experiments/rtp-hdrext/video-timing
	  a=extmap:8 http://www.webrtc.org/experiments/rtp-hdrext/color-space
	  a=extmap:4 urn:ietf:params:rtp-hdrext:sdes:mid
	  a=extmap:10 urn:ietf:params:rtp-hdrext:sdes:rtp-stream-id
	  a=extmap:11 urn:ietf:params:rtp-hdrext:sdes:repaired-rtp-stream-id
	  a=recvonly
	  a=rtcp-mux
	  a=rtcp-rsize
	  a=rtpmap:106 H264/90000
	  a=rtcp-fb:106 goog-remb
	  a=rtcp-fb:106 transport-cc
	  a=rtcp-fb:106 ccm fir
	  a=rtcp-fb:106 nack
	  a=rtcp-fb:106 nack pli
	  a=fmtp:106 level-asymmetry-allowed=1;packetization-mode=1;profile-level-id=42e01f
	  m=application 9 UDP/DTLS/SCTP webrtc-datachannel
	  c=IN IP4 0.0.0.0
	  a=ice-ufrag:D9lK
	  a=ice-pwd:Gq8blCpAHeOk23GNchayvj4o
	  a=ice-options:trickle renomination
	  a=fingerprint:sha-256 9E:65:BD:07:A7:F6:DD:05:A9:0F:23:88:8C:8F:AF:6A:64:30:C7:A3:6D:AC:2E:E5:16:D9:08:0B:55:A0:31:3F
	  a=setup:active
	  a=mid:2
	  a=sctp-port:5000
	  a=max-message-size:262144
	  ```
	- ## Subsequent candidate
-