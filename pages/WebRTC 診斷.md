public:: true

- ## 網路診斷
	- [[Emulate Network Conditions to Test WebRTC Adaptability]]
- ## 畫面模糊
	- **📌 畫面模糊 (Video Blurriness) 應該檢查哪些 WebRTC 指標？**
	- 如果 WebRTC 視訊出現模糊現象，可能是因為 **編碼壓縮過高、網路頻寬不足、解析度降低** 等因素。以下是你應該檢查的 **關鍵指標**：
	- ---
	- **1️⃣ 影像品質相關指標**
	  | 
	  **指標**
	  | 
	  **作用**
	  | 
	  **可能問題**
	  |
	  | ---- | ---- | ---- | ---- | ---- |
	  | 
	  **qpSum (量化參數總和)**
	  | 
	  **衡量視訊壓縮程度**，數值越高，壓縮越強，畫質越差
	  | 
	  高 QP 值 (如 40+) 代表畫面過度壓縮，可能導致模糊
	  |
	  | 
	  **framesPerSecond (FPS)**
	  | 
	  **檢查影格率是否過低**
	  | 
	  低 FPS (如 <15) 可能導致畫面看起來不流暢且模糊
	  |
	  | 
	  **frameWidth / frameHeight**
	  | 
	  **確認視訊解析度**
	  | 
	  解析度下降（如 1080p → 360p）通常表示網路問題
	  |
	  | 
	  **totalEncodeTime / framesEncoded**
	  | 
	  **計算編碼效率**
	  | 
	  如果編碼時間過長，可能導致延遲與壓縮問題
	  |
	- ---
	- **2️⃣ 網路相關指標**
	  | 
	  **指標**
	  | 
	  **作用**
	  | 
	  **可能問題**
	  |
	  | ---- | ---- | ---- | ---- | ---- |
	  | 
	  **targetBitrate (目標位元率)**
	  | 
	  **查看當前 WebRTC 設定的傳輸位元率**
	  | 
	  如果過低（如 <500kbps），可能導致影像過度壓縮
	  |
	  | 
	  **availableOutgoingBitrate**
	  | 
	  **發送端可用頻寬**
	  | 
	  頻寬低可能導致解析度下降
	  |
	  | 
	  **packetsLost (封包遺失數量)**
	  | 
	  **檢查是否有封包遺失**
	  | 
	  高遺失率可能造成畫面破損或模糊
	  |
	  | 
	  **jitter (網路抖動)**
	  | 
	  **檢查封包到達的穩定性**
	  | 
	  抖動高 (如 >30ms) 可能導致影像品質變差
	  |
	  | 
	  **retransmittedPacketsSent**
	  | 
	  **封包重傳次數**
	  | 
	  如果封包重傳次數過高，代表網路可能丟失了大量影像資料
	  |
	- ---
	- **3️⃣ 影像恢復與適應機制**
	  | 
	  **指標**
	  | 
	  **作用**
	  | 
	  **可能問題**
	  |
	  | ---- | ---- | ---- | ---- | ---- |
	  | 
	  **pliCount (Picture Loss Indication)**
	  | 
	  **檢查視訊是否因封包遺失而請求重新傳送**
	  | 
	  高 PLI 代表頻繁畫面丟失，導致畫面品質不穩定
	  |
	  | 
	  **firCount (Full Intra Request)**
	  | 
	  **檢查是否頻繁要求 I-frame (關鍵影格)**
	  | 
	  FIR 高可能表示畫面頻繁變糊並需要重新同步
	  |
	  | 
	  **totalPacketSendDelay**
	  | 
	  **發送封包的延遲**
	  | 
	  如果延遲過高，可能導致畫面品質下降
	  |
	- ---
	- **📌 如何快速診斷模糊問題？**
	- 1️⃣ **解析度是否降低？** 🔍
	- 👉 **檢查 frameWidth / frameHeight** 是否變小。
	- 👉 如果解析度下降，可能是 **頻寬不足 (availableOutgoingBitrate) 或 targetBitrate 設得過低**。
	- 2️⃣ **是否因壓縮過度導致模糊？** 🎥
	- 👉 **檢查 qpSum**：如果值過高 (>40)，代表影像壓縮過多，影像細節流失。
	- 👉 **檢查 totalEncodeTime / framesEncoded**：如果編碼時間過長，可能是編碼器處理不佳或 CPU 過載。
	- 3️⃣ **是否因網路問題導致模糊？** 🌐
	- 👉 **檢查 packetsLost、jitter**：高封包遺失或抖動可能導致畫面破損、模糊或畫面延遲。
	- 👉 **檢查 retransmittedPacketsSent**：如果重傳次數過多，表示網路封包丟失嚴重，導致影像不清晰。
	- 4️⃣ **是否頻繁請求 I-frame？** 🔄
	- 👉 **檢查 pliCount / firCount**：如果數值很高，代表視訊流頻繁因封包遺失而請求關鍵影格，影響畫質。
	- ---
	- **🎯 結論**
	- ✅ **如果解析度變低** → 檢查 **frameWidth / frameHeight、targetBitrate、availableOutgoingBitrate**
	- ✅ **如果壓縮過度** → 檢查 **qpSum (QP值)、totalEncodeTime / framesEncoded**
	- ✅ **如果網路不穩定** → 檢查 **packetsLost、jitter、retransmittedPacketsSent**
	- ✅ **如果頻繁請求 I-frame** → 檢查 **pliCount / firCount**
	- 你想要具體分析哪個 WebRTC 報告的數據嗎？🚀
- ## 卡頓