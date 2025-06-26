public:: true

- ### **Valid Multicast IP Range**
  
  Multicast IPs are in the range:
  
  ```
  224.0.0.0 to 239.255.255.255
  ```
  
  So 224.5.5.5 is valid. However:
	- Addresses in 224.0.0.0 - 224.0.0.255 are **reserved** for local network control and shouldn’t be used for general-purpose multicasting.
- ## 網路設定
	- Multicast group 是根據 IP join
	- AP 有 IGMP Snooping: 只有訂閱者收到 Multicast流量
	- AP 無 IGMP Snooping（Flood): 全部裝置都收到 => 整個Wi-Fi或Switch網路變慢爆
	- player 要選擇 wifi interface
	- player 有開 udp socket 和拿 multicast lock，AP 是 flood 的話會收到很多垃圾封包，需要做限流
		- **UDP socket 是否會收到資料，是根據「目的地 port（destination port）」來決定的，與 IP address（尤其 multicast IP）無關**。
			- 程式加入 group，但沒 bind 對 port：
			  結果：**OS 收到封包（因為 group 加入了）Wireshark 可以抓到，但 port 不對 → 不會 deliver 給這個 socket**
			- 程式沒有加入 group，但有綁 port：
			  結果：**OS 不會收到該 multicast 封包 → socket 根本沒機會拿到資料**
		- Multicast封包的 destination IP可以是任何224.x.x.x group。
		- 在 Flood模式下：
			- socket 只綁 port，所以只要 port符合，封包就送給 App。
			- 沒有 IP層級的自動 group filtering。
	- GStreamer udpsrc 會自動發 IGMP join
	- ### Socket bind
		- **multicast**
			- socket 要綁到 INADDR_ANY `bind(socket, "0.0.0.0:5004")`並且設定 `SO_REUSEADDR`、`SO_REUSEPORT` 讓多個 socket 可以使用
			- join(multicast IP, interface IP) 如果使用 join(multicast IP, INADDR_ANY) 作業系統會自動選一張
		- **unicast**
			- socket 綁到 interface local IP
		-
- # 頻寬估算
	- Protocol Overhead 加上封包的資訊、Header、加密 tag 等等約多出 15% 大小
	  
	  GPT 建議用原本的頻寬 * 1.3 來估
- ## 靜態內容
	- 約 2 Mbps、15 FPS
- ## 動態內容
	- 約 6.5 Mbps、30 FPS
	  
	  如果 AP 是使用 broadcast 發送，只會發出一次資料，接收設備數不影響頻寬，但 WiFi 會使用最慢的 multicast rate，使 airtime 長時間被佔用。
	  
	  當開啟 Multicast to Unicast 功能後流量和 airtime 都會乘上 N 倍，但每個封包都可重送，穩定性會提高。
- # 網路設備規格建議
	- 在同一個 VLAN 中使用 Multicast 傳輸時，封包會透過 Layer 2 Multicast 機制由交換器在 VLAN 內轉發，不需經過 Layer 3 routing，傳輸延遲較低、效率較高，適合大規模同步播放的場景。
	  
	  為確保穩定播放與良好延遲控制，建議 Wi-Fi AP 具備以下功能：
		- IGMP Snooping：協助交換器與 AP 僅將 Multicast 封包轉送至已加入 group 的裝置，避免 flooding。
		- Multicast to Unicast：提升可靠性與傳輸速率，避免使用低速率的 broadcast 傳送方式。
		- Multicast Rate 設定：可手動設定 Multicast 傳送速率（建議 ≥12 Mbps），避免預設 6 Mbps 造成 Airtime 壅塞。
		- WMM / QoS 分類：將影音流量標記為高優先級（Video），避免被低優先流量擠壓。
	- 在影音播放場景中，單台企業級 AP 實務上可穩定支援約 20～30 台裝置同時接收高畫質影片。若接收端超過此數量，建議部署多台 AP：
		- 多台 AP 橋接於同一 VLAN，維持 Layer 2 Multicast 可用，避免因 VLAN 隔離導致 Multicast 無法傳播。
		- 如果裝置位於不同 VLAN，需要在 L3 設備上設定 IGMP Proxy 或 PIM，以允許 Multicast 封包跨 VLAN 傳遞。
		- 進行頻道規劃與 Load Balancing，將裝置分佈至不同無線頻道與 AP，以避免 AP 間干擾與 Airtime 過度擁擠。