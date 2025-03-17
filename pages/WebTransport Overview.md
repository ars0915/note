# Background
	- **傳統雙向數據流方案**  WebSockets 的順序性導致的 Head-of-Line Blocking 問題影響低延遲應用的效能
		- **Head-of-Line Blocking（HOL Blocking）** 是指當數據流中的一個數據包（或訊息）因為某些原因（例如丟失、延遲或必須按順序處理）而被阻塞時，後續的數據包也無法繼續傳輸或處理，即使它們之間沒有直接的依賴關係。
			- **HTTP/2**：雖然它透過多路復用（multiplexing）允許多個流在同一個 TCP 連線上並行傳輸，但如果底層 TCP 連線發生 HOL Blocking，所有流仍然會被影響。
			- **HTTP/3**：基於 QUIC（UDP 之上的協議），解決了 TCP 的 HOL Blocking 問題，允許獨立的流在不同的 QUIC 連接內並行傳輸，即使某個流遇到問題，也不會影響其他流。
	- **現有替代方案**
		- WebRTC 依賴 ICE 和 userspace SCTP，導致採用率低
			- SCTP 通訊協定之多重串流特性有別於傳統的 TCP 通訊協定單一串流（single-stream），當端點建立連線時，可預先相互協調將要使用的串流數量，並且能夠將不同類型的資料分別以不同的串流傳輸
		- 多 WebSocket 連線的替代設計: 透過 HTTP/3 開啟多個 WebSocket 連線
			- 主要缺點：
				- 每個流都需要 WebSocket 握手，增加 RTT
				- 只能由客戶端啟動流，缺乏伺服器啟動的能力
				- WebSocket 連線通常會被「使用者代理（user agent）」集中管理，當應用程式開啟多個 WebSocket 連線時，**瀏覽器可能會嘗試將它們集中管理（pooled）**，讓它們在同一個 TCP 連線上共享資源。
				  但這不是標準化的行為，每個瀏覽器的實作方式可能不同。
				  **開發者無法確保所有 WebSocket 連線都會被集中在同一個 TCP 連線上。**
				  如果 WebSockets 被分配到不同的 TCP 連線，那麼這些連線之間可能會互相影響（如網路擁塞時），而這種影響是不可預測的。
	- **WebTransport 的優勢**
		- WebTransport 提供 **單一連線、多流（multiplexed streams）** 的設計，確保所有流在同一個連線內。
		- 開發者可以確保所有數據流共享同一個 QUIC 連線，避免 WebSocket 這種不可預測的行為。
		- **能夠優化不同數據流的傳輸優先級**，例如某些流可以設定為高優先級，某些流則可以允許丟包（類似 UDP）。
		- 提供不可靠數據報傳輸 (類似 UDP)
	-
- ## Reference
- [https://datatracker.ietf.org/doc/draft-ietf-webtrans-overview/](https://datatracker.ietf.org/doc/draft-ietf-webtrans-overview/)
- [https://datatracker.ietf.org/doc/html/draft-ietf-webtrans-http3/#name-quic-webtransport-and-http-](https://datatracker.ietf.org/doc/html/draft-ietf-webtrans-http3/#name-quic-webtransport-and-http-)
- https://sites.google.com/site/applezulab/network/sctp_introduction
-