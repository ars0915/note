public:: true

- ## Wi-Fi P2P
	- discovery 階段有以下狀態
		- Scan： 在所有 channel 找有沒有人
		- Find
			- Listen： 監聽某個 channel，接到 request 時回應
			- Search：針對一個 list 的 social channel 們發送 request
	-
- ## Mice
	- over infra，比 P2P 穩定
	- 透過 mDNS 拿到 AP 分配的 IP
- ### Trouble shooting
	- Android -> Wifi verbose logging
	- 看 wpa_supplicant 層觀察操作 Wi-Fi 網卡的 log
		- wpa_cli