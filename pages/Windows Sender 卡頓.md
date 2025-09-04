# Stats 觀察
	- packetSendDelayAvgMs 很高 （1000.83,1893.87,1875.24,2000.34,1737.83,1948.83,1828.76,1501.58,1259.65,740.78）
	- framesEncodedPerSecond 很低（5,7,8,7,7,8,8,8,8,8）
	- bytesSentPerSecond 一開始低（116272,157196,193374,195622,246239,296953,316706,351522,407036,476824）
- # 可能的原因
	- ## BWE（Bandwidth Estimation）錯估太低
	  常見於 TWCC 沒有生效 或 回報異常
	- ## 幀太大 / 碼率突發
	  例如關鍵幀（IDR frame）或場景切換，突然 500KB～1MB
	- ## 發送端執行緒排程壅塞
	- ## RTP 分片太多
- # 需要增加 log
	- ## availableOutgoingBitrate
		- 和實際碼率對比
			- 觀察同時間的：
				- availableOutgoingBitrate
				- bytesSentPerSecond × 8（實際碼率）
			- 正常情況：
				- 實際碼率 ≈ availableOutgoingBitrate（稍微低一點，因為發送端要留 margin）
			- 異常情況：
				- availableOutgoingBitrate 明顯比實際碼率低很多（例：available=0.5 Mbps，但實際還在送 3 Mbps） → TWCC/RTCP 可能失效，BWE 沒被正確採用
				- availableOutgoingBitrate 長時間卡在極低值（例如 30–50 kbps），畫面卻完全卡住 → BWE 錯估
		- 隨時間變化
			- 正常：
				- 起始時小（例如 300 kbps），幾秒內快速「ramp up」到多 Mbps，之後穩定。
			- 異常：
				- 長時間不上升（例如一直卡在起始值）
				- 或忽上忽下（1 Mbps ↔ 100 kbps ↔ 1 Mbps ↔ …），但網路其實穩 → 算法沒正確收到回報
		- 和 packetSendDelayAvgMs、jitterBufferDelay 對照
		  id:: 68b94627-f11e-44c3-8a6c-9933138b6b0e
			- 如果 availableOutgoingBitrate 太低 → sender 應該自動降碼率 → queue 不應該爆。
			- 如果你看到：
				- availableOutgoingBitrate 很低
				- 但 sender 還在高碼率送(看bytesSentPerSecond） → packetSendDelayAvgMs 飆高 → jitter buffer 堆積
				  -> 這表示 sender 沒有用到 BWE（可能沒啟用 TWCC feedback）
	- ## 如何判斷幀太大
		- hugeFramesSent：大幀計數（libwebrtc 會標記異常大的 frame）
		- framesEncoded / framesSent：幀數
		- bytesSent：總傳輸位元組數
		- 如果 hugeFramesSent 有增加、bytesSent 瞬間暴衝（遠高於平均）-> 表示有超大幀被送出（通常是 IDR/關鍵幀）。
		- qpSum：如果平常幀的 QP 在 30 附近，突然掉到 18 → 表示編碼器產生了很大、很高畫質的幀（常見於 IDR）。
		- ### 疑似 IDR 時刻
			- ΔkeyFramesEncoded > 0
			- ΔhugeFramesSent > 0
			- 觀察某個區間 bitrate_bps > 滾動中位數的 3–5 倍（突發碼率尖峰）
		- ### 判斷「是否異常大」
			- 用同一區間的 availableOutgoingBitrate 當分母，估計「送出這個區間新增位元組需要多久」：
			  idr_send_time_est = 8*(ΔbytesSent) / availableOutgoingBitrate
			  	•	用同一區間的 fps 算幀間隔：frame_interval = 1/fps
			  	•	若 idr_send_time_est > 2 × frame_interval ⇒ 偏異常
			  （>1× 已吃緊，>2× 高機率撐爆 pacer/queue）
			- 同一區間的 pktSendDelay_ms 明顯升高（例如 > 50–100 ms，或較前 5 個區間中位數高出 3 倍），且與上面 IDR 訊號同時發生
			  ⇒ 高機率是 IDR 太大導致的壅塞。
		- ### 交叉驗證（網路 & BWE 是否合理）
			- 若同一時間 availableOutgoingBitrate 沒掉太多，但 bitrate_bps 卻暴衝、pktSendDelay_ms 升高
			  ⇒ 不是網路突降，是幀本身過大或 pacer 不平滑。
			- 若 availableOutgoingBitrate 本來就很低（例如 < 1 Mbps），任何 IDR 都容易變「相對過大」：優先降低 keyframe 大小/頻率或先把 BWE 問題解掉。
	- ## 估 RTP 分片
		- 在每個取樣區間（Δt）計算：
			- 每幀封包數（PPF）
			  `PPF ≈ (ΔpacketsSent) / (ΔframesEncoded 或 ΔframesSent)`
			- 每包平均有效載荷（APL）
			  `APL_bytes ≈ (ΔbytesSent) / (ΔpacketsSent)`
		- 內網/一般 MTU 下，扣掉 IP/UDP/RTP 頭，單包 1000–1200B 有效載荷很常見。
		- Inter 幀
			- 基準：PPF_med_inter（最近 3–5 秒中位數）
			- 若長時間 PPF > 2 × PPF_med_inter，且 APL < ~700–900B ⇒ 高機率「分片過細」或「很多迷你包」
		- Keyframe（IDR）
			- 在 ΔkeyFramesEncoded > 0 的區間，若
				- PPF_idr > 50（1080p 常見上限；>100 幾乎必然偏多），或 APL_idr 明顯 < 目標（例如 <800B）⇒ 這個 IDR 的 RTP 分片/打包顆粒度偏不理想（包太小、包太多）
			- 連動訊號
				- 同時 ΔtotalPacketSendDelay/ΔpacketsSent（區間平均送包延遲）在 keyframe 區間突升 ⇒ 過多分片 + 短時突發，正在撐爆 pacer/queue
		- 取最近 3–5 秒的 inter 幀 PPF 與 APL 的中位數做基準。
		- 檢查「keyframe 區間」與「一般區間」：
			- 一般區間：若 PPF 長期偏高且 APL 偏小 ⇒ 分片過多（即使沒有 keyframe）。
			- keyframe 區間：若 PPF 明顯尖峰（>50 或 >基準 8–12 倍）且 packetSendDelay 同步升 ⇒ IDR 分片/打包不佳。
		- 若同時看到 availableOutgoingBitrate 正常、實際碼率 ≈ 其值，但 packetSendDelay 在 keyframe 區間飆高 ⇒ 不是 BWE 問題，是分片/幀太大問題。
	- ## Capture → Encoder 有冇 drop
		- media-source（或 video-source）
			- frames（有些實作叫 framesCaptured）
			- framesPerSecond
		- outbound-rtp（video）
			- framesEncoded、totalEncodeTime、keyFramesEncoded、hugeFramesSent
		- `ΔframesCaptured  - ΔframesEncoded  >> 0`  ⇒ Capture→Encoder 之間有 drop
	-
		-