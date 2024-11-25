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
	  1. SDP offer 初始化時可能會帶個 private IP 或 default 值
	  2. 直連時會是 media 交換的 IP
	- `a=rtcp:51472 IN IP4 217.130.243.155`: rtcp port, net type, addr type, connection address
	  **RTCP** 與 RTP 一起使用來監控 、報告統計資料以及管理音訊和視訊之間的同步。
		- In modern WebRTC, this line is often superseded or supplemented by:
			- ICE candidates provide the actual transport addresses for RTP and RTCP after negotiation, including NAT traversal.
			- RTCP Multiplexing:
			  1. WebRTC frequently uses RTCP multiplexing (rtcp-mux), which combines RTP and RTCP packets on the same port to simplify the connection and reduce resource usage.
			  2. If rtcp-mux is used, you may see a line like a=rtcp-mux, and the a=rtcp line might be omitted or ignored.
	- ## ICE Candidates
		- `a=candidate:1467250027 1 udp 2122260223 192.168.0.196 46243 typ host generation 0`:
		  
		  ```shell
		  a=candidate:<foundation> <component-id> <transport> <priority> <address> <port> typ <type> [<related-address> <related-port>] [<extensions>]
		  ```
			- component ID: `1`: RTP, `2`: RTCP
			- priority: 值愈大代表 priority 愈高，is determined by factors such as type (e.g., host vs. relay), network reachability, and other ICE-specific metrics
			- typ: `host`: directly with local network interface (e.g. private IP), `srflx`: server reflexive, 經過 STUN 拿到的 public IP, `relay`: TURN server, [[prflx]]: peer reflexive, Discovered during ICE connectivity checks.
	- ## ICE Parameters
		- `a=ice-ufrag:Oyef7uvBlwafI3hT`: ICE username fragment.
		  It is paired with the **ICE password** (`a=ice-pwd`) to authenticate and match connectivity checks between peers.