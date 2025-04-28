public:: true

- 在傳輸層
  UDP: 傳輸資料
  TCP: 可靠的傳輸資料
  RTP: 傳輸多媒體資料
- ## Header
	- CSRC: 記錄所有參與方的來源 SSRC
	- SSRC: 記錄封包的發送方，隨機從 md5 選取，在同一個視訊中不會有相同的 SSRC
	- ![image.png](../assets/image_1745822962484_0.png)
-