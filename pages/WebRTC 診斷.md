public:: true

- ## 網路診斷
	- [[Emulate Network Conditions to Test WebRTC Adaptability]]
- ## 畫面模糊
	- **📌 畫面模糊 (Video Blurriness) 應該檢查哪些 WebRTC 指標？**
		- 如果 WebRTC 視訊出現模糊現象，可能是因為 **編碼壓縮過高、網路頻寬不足、解析度降低** 等因素
	- **📌 如何快速診斷模糊問題？**
		- 1️⃣ **解析度是否降低？** 🔍
			- 👉 **檢查 frameWidth / frameHeight** 是否變小。
			  👉 如果解析度下降，可能是 **頻寬不足 (availableOutgoingBitrate) 或 targetBitrate 設得過低**。
		- 2️⃣ **是否因壓縮過度導致模糊？** 🎥
			- 👉 **檢查 qpSum**：如果值過高 (>40)，代表影像壓縮過多，影像細節流失。
			  👉 **檢查 totalEncodeTime / framesEncoded**：如果編碼時間過長，可能是編碼器處理不佳或 CPU 過載。
		- 3️⃣ **是否因網路問題導致模糊？** 🌐
			- 👉 **檢查 packetsLost、jitter**：高封包遺失或抖動可能導致畫面破損、模糊或畫面延遲。
			  👉 **檢查 retransmittedPacketsSent**：如果重傳次數過多，表示網路封包丟失嚴重，導致影像不清晰。
		- 4️⃣ **是否頻繁請求 I-frame？** 🔄
			- 👉 **檢查 pliCount / firCount**：如果數值很高，代表視訊流頻繁因封包遺失而請求關鍵影格，影響畫質。
	- **🎯 結論**
		- ✅ **如果解析度變低** → 檢查 **frameWidth / frameHeight、targetBitrate、availableOutgoingBitrate**
		  ✅ **如果壓縮過度** → 檢查 **qpSum (QP值)、totalEncodeTime / framesEncoded**
		  ✅ **如果網路不穩定** → 檢查 **packetsLost、jitter、retransmittedPacketsSent**
		  ✅ **如果頻繁請求 I-frame** → 檢查 **pliCount / firCount**
- ## 卡頓
	- 📌 步驟 1：影像是否有明顯掉幀？
		- 👉 檢查 framesDropped、framesPerSecond、framesDecoded
		  👉 如果 framesDropped 高，代表解碼端無法處理影像，可能是 CPU 過載 或 網路問題。
	- 📌 步驟 2：網路是否有明顯異常？
		- 👉 檢查 packetsLost、jitter、totalPacketSendDelay
		  👉 如果 packetsLost 過高，代表封包遺失，可能導致畫面停頓。
		  👉 如果 jitter > 30ms，代表封包到達不穩定，影像播放可能不順暢。
	- 📌 步驟 3：視訊是否明顯延遲？
		- 👉 檢查 roundTripTime、pliCount
		  👉 如果 roundTripTime > 300ms，代表視訊可能與音訊不同步，或影像延遲。
		  👉 如果 pliCount 高，代表頻繁請求 I-frame，影像可能不穩定。
	- 🎯 結論
		- ✅ 如果是影像掉幀 → 檢查 framesDropped、framesPerSecond，避免 CPU 過載
		  ✅ 如果是網路卡頓 → 檢查 packetsLost、jitter、retransmittedPacketsSent，確保頻寬足夠
		  ✅ 如果是視訊延遲 → 檢查 roundTripTime、totalInterFrameDelay，優化網路與封包傳輸