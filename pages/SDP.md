# Session-Level
	- `o=- 4611731400430051336 2 IN IP4 127.0.0.1`: session id, session version, network type
	  session version 在 SDP negotiation 時都會 +1
	- `s=-`: session name (often unused)
	- `t=0 0`: start and end time (often unused)
- # Group Attributes
	- `a=group:BUNDLE audio video`: The BUNDLE mechanism allows multiple media streams to share the same network connection (port and transport), which reduces resource usage.
	- `a=msid-semantic: WMS lgsCFqt9kN2fVKw5wg3NKqGdATQoltEwOdMS`: 
	  表示這個 SDP 用 Media Stream Identification (MSID) semantic
	  The **WMS** keyword refers to "WebRTC Media Streams," a concept in WebRTC where media tracks are grouped into a single logical stream for easier handling.
	  `lgsCFqt9kN2fVKw5wg3NKqGdATQoltEwOdMS` 為 ID，讓 application 管理 WebRTC MediaStream object
- # Media Descriptions
	- `m=audio 54321 UDP/TLS/RTP/SAVPF 111 103 104`: media type, including port, protocol, supported codec
	- `c=IN IP4 217.130.243.155`: net type, address type, connection address
	  定義在 top-level 的 `c=` 會 apply 到所有 media section (`m=` lines)，當 media section 有定義自己的 `c=` line 會 override global `c=` line
	  address 可能的值
	  1. SDP offer 初始化時可能會帶個
	-