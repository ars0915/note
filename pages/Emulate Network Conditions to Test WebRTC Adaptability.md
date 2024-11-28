public:: true

- # Normal
	- ## DTLS handshake
	  ![image.png](../assets/image_1732778116493_0.png)
	- ## STUN and TURN
	  ![image.png](../assets/image_1732778427697_0.png)
	- ## RTP media
		- ### How to find filter `ip.addr==172.21.13.157 && udp.port==62481 && ip.addr==172.21.10.181 && udp.port==43716`
			- **From chrome tool**
				- ICE Candidate pair in `chrome://webrtc-internals`
			- **From WireShark**
				- Navigate to: Statistics > Conversations.
				  Focus on UDP Conversations
				  In the Conversations window, switch to the UDP tab.
				  This tab shows a list of all UDP conversations (source/destination IP and port pairs) along with statistics like packet count, byte count, and duration.
				- Identify Streaming Traffic
				  Look for streams with a high packet count and a steady byte rate—typical for streaming traffic.
				  If you know the IP address of the streaming server, look for conversations involving that IP.
				  Key Indicators of Streaming Traffic:
			- High packet count: Indicates a sustained stream of data.
			  Consistent data size: Streaming often uses similar-sized packets.
			  Known server IPs: Check if you recognize streaming services like Netflix, YouTube, or others.
			  5. Filter Specific UDP Stream
			  Select the relevant conversation and note its source and destination IP addresses and port numbers.
			  Click Apply as Filter to focus on that specific UDP stream.
			  Example filters:
			- Filter by IP addresses and ports:
	- ## P2P Connections
	  ![image.png](../assets/image_1732779164495_0.png)
	-
- # Blocking Direct Connections
-
-