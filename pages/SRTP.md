# 要解決的問題
- 對 RTP / RTCP 的 payload 加密
- 保證 RTP / RTCP 的完整性，同時防止重放攻擊
- # 結構
- ![image.png](../assets/image_1745823294705_0.png)
	- Encrypted Portion 由 payload, RTP padding, RTP pad count 組成
	- 需要驗證部份 Authenticated Portion 由 Encrypted Portionm, RTP header, RTP header extension 組成
	- 通常只對 payload 加密
- # Key 管理
	- Session: <SRTP 目的 IP, SRTP 目的 port>
	- Stream: <SSRC, RTP/RTCP 目的IP, RTP 目的 port>，一個 SRTP / SRTCP Session 由多個 stream 組成。對每個 stream 的加解密相關參數描述為 Cryptographic Context。
	- 每個 stream 的 Cryptographic Context 中包含以下參數
		- SSRC
		- Cipher Parameter: 加解密使用的 key, salt，算法類型，參數
		- Authentication Parameter: 完整性使用的 key, salt，算法類型，參數
		- Anti-Replay Data: 防止重放攻擊緩存的數據信息
	- 在 Session 中每個 Stream 都會用到屬於自己的加解密 key, Authentication key。這些 Session key 是通過對 Master key 使用 KDF (Key Derivation Function) 導出
	- ![image.png](../assets/image_1745824069494_0.png)
	- master key: DTLS 完成後協商得到的 key
	- master salt: DTLS 完成後協商得到的 key
- # 序列號
	- RTP 封包內的序列號
	  	•	每發送一個 RTP 封包，序列號加 1。
	  	•	當序列號達到 65535 時，回繞（wrap around）回到 0。
	  	•	這個序列號放在 RTP 封包頭中，占 16 bits。
	- SRTP 封包保護（加密/認證）
	  	•	加密和認證時，序列號會被用來產生 IV (Initialization Vector)。
	  	•	IV 通常是基於：
	  	•	SSRC（Synchronization Source）
	  	•	序列號
	  	•	其他密鑰派生參數
	  	•	這樣設計可以確保即使資料內容重複，因為序列號不同，最終加密結果也不同，增加安全性。