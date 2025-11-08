- 排查問題要通靈 -> 缺乏系統性的 observability，無法用工程方法解決問題，只能靠經驗和運氣
  id:: 690f0f53-0590-4c93-abd6-fc232dead237
	- 許多因素來自系統，但沒有系統、當下環境的 log，根本不知道發生什麼事
	- 靠複現問題很花時間，也不全面
	- 使用加 log 的方式排查只是針對已知問題
- 時程規劃混亂
	- sprint 的安排跟實際常有出入，做到一半就要包版
	- sprint 中間想插單就插
	- 每個客戶反應的問題就 P1，無止盡的 P1，修完一個又一個，一直處在這狀況很 burnout -> 導致無法專注做深度優化和系統性改善
	- 每次 sprint 只是在做個形式，重點只在表面上單要不要關，要不要拆開排到下一個，實際上和 scrum 的規劃精神違背
	- 急著發版不先考量做出來的修正是否造成 side effect，有問題又堅持不退版，卡在進退兩難不斷修新 bug 的狀況
	- 惡性循環：沒 monitoring → 問題難排查 → 花更多時間救火 → 沒時間建立 monitoring
- 硬體相關的問題
	- 換個板子或 FW 版本就又有新問題，花很多時間在測試，但又測的不全面
	- 在 intranet, Wifi P2P 和客戶的裝置、環境有很大關係，在開發或測試時很難考量到，只能遇到問題又通靈
	- 大屏本身問題很多，還要根據晶片等等做 workaround
- Streaming -> 缺乏客觀的品質指標和測量標準，難以用工程方法優化
	- 影像品質很主觀，連描述問題都沒辦法統一，到底是卡頓、還是跳幀、還是tearing 等等
	- 一直在改動看起來的感覺，解析度, bitrate, FPS 改來改去，跟前端需要一直調感覺 87% 像
	- 沒有 trade-off 的觀念，要快要順又要清晰高畫質
- 個人因素
	- 一年來感覺自己只是 AI 傳聲筒，學到的東西很片段
	- 看有能力的同事們工作仍然很辛苦，不想未來過一樣的生活
	- 有興趣的地方比較偏向網路、演算法相關，對於怎麼 render 之類的沒什麼興趣
	- 希望在更注重工程實踐和可測量性的環境工作
	- 主管態度消極，對於升遷、向公司申請 AI 工具都在推拖，覺得在這裡沒未來
- ## 總結
	- ### 類別 1：系統性問題（最重要）
		- 缺乏 observability 和 monitoring 系統
		- 無法用工程方法系統化解決問題
		- Management 不願改善開發流程
	- ### 類別 2：團隊文化問題
		- 時程規劃混亂，缺乏專業的 sprint 管理
		- 一直在救火，無法做深度優化
		- 主管態度消極，看不到改善空間
	- ### 類別 3：個人發展不符
		- 工作內容與專長不符（太多硬體整合）
		  Client side 的 streaming 有很多跟系統高度相關的部份，想做偏向 Server-side 的部份
		- 缺乏技術成長空間
		  比較像是 AI 傳聲筒，解決問題靠經驗跟運氣
		- 想要更系統化的技術挑戰
		- 我喜歡的
		  Server-side streaming infrastructure:
		  ├── P2P signaling server (Mlytics) 
		  ├── SFU server 除錯 (ViewSonic)
		  ├── WebRTC protocols
		  ├── Real-time connection management
		  ├── Scalability challenges (100K connections)
		  └── Distributed systems design
		- 不喜歡的
		  Client-side implementation:
		  ├── MediaCodec/MediaProjection 處理
		  ├── OpenGL rendering pipeline
		  ├── Platform API 限制和 workaround
		  ├── 硬體整合問題
		  └── 各種板子的 bug
-
- ## 1. 產品定位不符合專長和興趣
- **現況**：產品以硬體為主，大量時間處理 client-side media pipeline 和硬體整合
- **困擾**：
	- Client-side: MediaCodec/MediaProjection recovery, OpenGL rendering optimization
	- 硬體: 不同晶片/FW 版本的問題，需要軟體 workaround
	- 平台適配: Android/iOS/ChromeOS 的 API 限制和差異
- **真正興趣**：Server-side streaming infrastructure
	- P2P signaling server (Mlytics: 100K concurrent connections)
	- SFU/MCU server 架構設計和優化
	- WebRTC protocols 和 real-time communication
	- Distributed systems scalability
- ## 2. 缺乏工程實踐和可測量性
- **現況**：
	- 缺乏 observability 系統，無法系統化排查問題
	- 影像品質缺乏客觀指標，難以量化優化效果
	- Management 不支持導入 proper monitoring tools
- **影響**：
	- 只能靠經驗和運氣排查問題（"通靈"）
	- 無法從根本解決問題，一直在打補丁
	- 無法做系統性的性能優化和改善
	- ## 3. 時程規劃和開發流程問題
	- Sprint 規劃混亂，常中途插單或提前包版
	- 無止盡的 P1 bugs，一直在救火
	- 急著發版不考慮 side effects，導致惡性循環
	- 無法專注做深度優化和系統性改善
	- ## 4. 個人成長受限
	- 工作太瑣碎，學到的東西很片段化
	- 一直在救火，沒時間研究新技術或深入優化
	- 主管態度消極，看不到改善空間和未來
	- 想要在更注重工程實踐的環境工作
	-
	-
	-
	-
	-
-
-