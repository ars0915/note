public:: true

- ### **Valid Multicast IP Range**
  
  Multicast IPs are in the range:
  
  ```
  224.0.0.0 to 239.255.255.255
  ```
  
  So 224.5.5.5 is valid. However:
	- Addresses in 224.0.0.0 - 224.0.0.255 are **reserved** for local network control and shouldn’t be used for general-purpose multicasting.
- ## 網路設定
	- AP 有 IGMP Snooping: 只有訂閱者收到 Multicast流量
	- AP 無 IGMP Snooping（Flood): 全部裝置都收到 => 整個Wi-Fi或Switch網路變慢爆
	- player 要選擇 wifi interface
	- player 有開 udp socket 和拿 multicast lock，AP 是 flood 的話會收到很多垃圾封包，需要做限流
		- UDP socket用的是「目的 port」來決定 delivery，不是靠 IP address。
		- Multicast封包的 destination IP可以是任何224.x.x.x group。
		- 在 Flood模式下：
			- socket 只綁 port，所以只要 port符合，封包就送給 App。
			- 沒有 IP層級的自動 group filtering。