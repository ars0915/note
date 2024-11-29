public:: true

- # Normal
	- ## DTLS handshake
	  ![image.png](../assets/image_1732778116493_0.png)
	- ## STUN and TURN
	  ![image.png](../assets/image_1732778427697_0.png)
	- ## RTP media
		- ### How to find filter `ip.addr==172.21.13.157 && udp.port==62481 && ip.addr==172.21.10.181 && udp.port==43716`
			- #### **From chrome tool**
				- ICE Candidate pair in `chrome://webrtc-internals`
			- #### **From WireShark**
				- **Capture Network Traffic**
					- Start capturing packets while the streaming application is running.
					- Use a filter (optional) to focus on UDP traffic during capture
				- **Open the Conversations Window**
					- Stop the capture once you have sufficient traffic.
					- Navigate to: **Statistics** > **Conversations**.
				- **Focus on UDP Conversations**
					- In the **Conversations** window, switch to the **UDP** tab.
					- This tab shows a list of all UDP conversations (source/destination IP and port pairs) along with statistics like packet count, byte count, and duration.
					  ![image.png](../assets/image_1732789740137_0.png)
				- **Identify Streaming Traffic**
					- Look for streams with a **high packet count** and a **steady byte rate**—typical for streaming traffic.
					- If you know the IP address of the streaming server, look for conversations involving that IP.
					  **Key Indicators of Streaming Traffic:**
						- High **packet count**: Indicates a sustained stream of data.
						- Consistent **data size**: Streaming often uses similar-sized packets.
			- ### Decoded as RTP
				- Select the packet then right click `Decoded As...`
				  Set Current to `RTP`
				  ![image.png](../assets/image_1732789923823_0.png)
			- ### Check info
			  seq 會是遞增
			  PT 的 codec 會在 SDP 中找到 (e.g., 106)
			- ![image.png](../assets/image_1732789968816_0.png){:height 368, :width 771}
	- ## P2P Connections
	  ![image.png](../assets/image_1732779164495_0.png)
	-
- # Blocking Direct Connections
	- macOS use `pfctl` to block traffic
		- ```
		  sudo vim /etc/pf.conf
		  ```
		- At the end of the file add the following to block UDP on port 12345
		  ```
		  block quick proto udp from any to any port = 12345
		  ```
		- Then enable the firewall with
		  ```
		  sudo pfctl -f /etc/pf.conf && sudo pfctl -E
		  ```
	- ### Web sender
		- Block port after connected
			- 出現 Reconnecting 後斷線
		- Block target IP after connected
		  ```
		  block quick proto udp from any to 172.21.10.181
		  ```
			- 使用 TURN
			  ![image.png](../assets/image_1732852666345_0.png)
-
-