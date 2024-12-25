# Problem
	- Chromebook 使用 Android sender 抓不到系統聲音
	- 使用 web sender 只能透過 internet
- # 現狀
	- encode IP 後 3 碼得到`組別`
	- 將`組別` + instanceID 跟 Backend 註冊得到`序號`
	- **internet**
	  透過`組別`、`序號`經過 Backend 轉發配對
	- **intranet**
	  native sender 透過查自己 IP 假設第1碼一樣(內網IP網段可能是 192、172、10)
- # Goal
	- ## Web sender 包裝成 PWA
	- ## Web sender 支援 intranet
		- Connect to a self-signed local server without a browser warning