public:: true

- ## inbound
	- ```
	  12 = {StatsReport} 
	   id = "IT01V1928484940"
	   type = "inbound-rtp"
	   timestamp = 1742263372529415.0
	   values = {_Map} size = 41
	    0 = {map entry} "qpSum" -> 20695
	    1 = {map entry} "lastPacketReceivedTimestamp" -> 1742263372478.0
	    2 = {map entry} "transportId" -> "T01"
	    3 = {map entry} "mid" -> "0"
	    4 = {map entry} "nackCount" -> 26
	    5 = {map entry} "googTimingFrameInfo" -> "2893125860,-65,-59,-45,-44,-1,-65,-65,1295254453,1295254454,1295254520,1295255042,1295255244,0,1"
	    6 = {map entry} "firCount" -> 0
	    7 = {map entry} "pliCount" -> 0
	    8 = {map entry} "packetsLost" -> 44
	    9 = {map entry} "freezeCount" -> 5
	    10 = {map entry} "totalFreezesDuration" -> 3.537
	    11 = {map entry} "keyFramesDecoded" -> 1
	    12 = {map entry} "pauseCount" -> 0
	    13 = {map entry} "totalSquaredInterFrameDelay" -> 28.500540000000033
	    14 = {map entry} "framesDropped" -> 110
	    15 = {map entry} "totalAssemblyTime" -> 64.27197799999999
	    16 = {map entry} "frameHeight" -> 1080
	    17 = {map entry} "framesAssembledFromMultiplePackets" -> 588
	    18 = {map entry} "decoderImplementation" -> "c2.android.avc.decoder (fallback from: OMX.sprd.h264.decoder)"
	    19 = {map entry} "powerEfficientDecoder" -> true
	    20 = {map entry} "framesReceived" -> 714
	    21 = {map entry} "kind" -> "video"
	    22 = {map entry} "trackId" -> "DEPRECATED_TI3"
	    23 = {map entry} "jitterBufferDelay" -> 510.757
	    24 = {map entry} "ssrc" -> 1928484940
	    25 = {map entry} "mediaType" -> "video"
	    26 = {map entry} "minPlayoutDelay" -> 0.0
	    27 = {map entry} "trackIdentifier" -> "C8B1E4BB-247C-449E-BC82-1D3DFF19B35D"
	    28 = {map entry} "totalPausesDuration" -> 0.0
	    29 = {map entry} "headerBytesReceived" -> 140480
	    30 = {map entry} "jitterBufferEmittedCount" -> 709
	    31 = {map entry} "codecId" -> "CIT01_98_level-asymmetry-allowed=1;packetization-mode=1;profile-level-id=42e01f"
	    32 = {map entry} "bytesReceived" -> 6095071
	    33 = {map entry} "totalProcessingDelay" -> 346.80222399999997
	    34 = {map entry} "jitter" -> 0.115
	    35 = {map entry} "frameWidth" -> 1920
	    36 = {map entry} "packetsReceived" -> 5546
	    37 = {map entry} "totalDecodeTime" -> 212.35899999999998
	    38 = {map entry} "framesPerSecond" -> 3.0
	    39 = {map entry} "framesDecoded" -> 602
	    40 = {map entry} "totalInterFrameDelay" -> 110.464
	  ```
	- 網路相關
		- packetsLost
		- framesDropped
		- pliCount: Number of Picture Loss Indication (PLI) messages received, signaling video loss recovery requests.
		  id:: 67dce088-e75d-44d3-8fbc-deecb820c2fd
		- framesReceived
		- jitterBufferDelay
		- jitterBufferEmittedCount
	- 影像品質相關
		- qpSum: 愈低 量化程度較低 → 更多細節保留 → 影像品質較高
		- frameHeight、frameWidth
		- key frame
			- keyFramesReceived: Keyframes received by the receiver.
			- firCount: Number of Full Intra Request (FIR) messages received, indicating how often keyframes were requested.
			- keyFramesDecoded
	-
	-
	-
	-
	-