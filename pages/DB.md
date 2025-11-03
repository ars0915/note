## Index 原理與優化：
	- B+ Tree 為什麼適合做索引？
		- 葉子節點有指針連接，適合範圍查詢
		- 非葉子節點只存 key，能存更多索引
		- 高度低（通常 3-4 層），IO 次數少
		- #### B+ Tree 的優勢
			- **1. 節點可以有多個子節點（不只 2 個）**
			  ```
			           [10, 20, 30]           ← 一個節點可以有 4 個指針
			          /    |    |    \
			    [1-9]  [10-19] [20-29] [30-39]
			  ```
			  
			  **一個節點能存多少 key？**
			  ```
			  假設：
			  - 一個 Page = 16KB（MySQL 的 InnoDB）
			  - 每個 key + pointer = 14 bytes
			  
			  一個節點可以存：16KB / 14B ≈ 1170 個 key
			  
			  → 高度只需要 3-4 層就能存千萬筆資料！
			  ```
			- **2. 所有資料都在葉子節點**
			  ```
			  非葉子節點：只存 key（用來導航）
			            [10 | 20 | 30]
			           /    |    |    \
			  葉子節點：   存 key + data
			         [1,data] [2,data] ... 
			         ↔ 用指針連接 ↔
			  ```
	- 什麼時候索引會失效？
		- 使用 `!=`、`<>`、`OR`可能直接**全表掃描**反而更快
		- Like 開頭用 `%`
		- 對索引欄位做函數運算
		- 隱式類型轉換
	- 聯合索引的最左匹配原則
		- `(a, b, c)` 的索引只能用於 `a`、`a,b`、`a,b,c` 的查詢
		- **規則：索引只能從最左邊開始連續使用**
			- 能用到索引的查詢：
			  ```sql
			  -- Case 1: 用 a （✅ 用到索引）
			  SELECT * FROM users WHERE a = 1;
			  
			  -- Case 2: 用 a, b （✅ 用到索引）
			  SELECT * FROM users WHERE a = 1 AND b = 2;
			  
			  -- Case 3: 用 a, b, c （✅ 用到索引）
			  SELECT * FROM users WHERE a = 1 AND b = 2 AND c = 3;
			  
			  -- Case 4: 順序無關 （✅ 用到索引）
			  SELECT * FROM users WHERE b = 2 AND a = 1;  -- MySQL 會優化
			  ```
			- 不能用到索引的查詢：
			  ```sql
			  -- Case 1: 跳過 a，直接用 b （❌ 不能用索引）
			  SELECT * FROM users WHERE b = 2;
			  -- 原因：索引先按 a 排序，沒有 a 條件就無法定位
			  
			  -- Case 2: 跳過 a，用 c （❌ 不能用索引）
			  SELECT * FROM users WHERE c = 3;
			  
			  -- Case 3: 用 a 和 c，跳過 b （⚠️ 只能用到 a 的索引）
			  SELECT * FROM users WHERE a = 1 AND c = 3;
			  -- MySQL 會用 a 的索引定位，然後逐行檢查 c
			  ```
	- ### 面試常見問題
		- **Q1: `WHERE a = 1 AND c = 3` 能用到索引嗎？**
			- A: 只能用到 `a` 的索引，`c` 無法用索引。
				- MySQL 會用 `a=1` 定位到一組資料
				- 然後逐行檢查 `c` 是否等於 3（table scan）
		- **Q2: 為什麼要遵守最左匹配？**
			- A: 因為聯合索引是**按順序排序**的。跳過前面的欄位，後面的就不是有序的，無法用索引。
		- **Q3: 如何優化 `WHERE b = 2` 的查詢？**
			- A: 兩種方式：
				- 建立單獨的索引：`CREATE INDEX idx_b ON users (b);`
				- 建立新的聯合索引：`CREATE INDEX idx_bc ON users (b, c);`
	- ### 實戰技巧
		- **如何選擇聯合索引的順序？**
			- 等於條件放前面，範圍條件放後面
			  ```sql
			  -- 如果查詢是：WHERE a = 1 AND b > 2
			  -- 索引應該建：(a, b) 而不是 (b, a)
			  ```
			- 選擇性高的放前面
			  ```sql
			  -- user_id 有 100 萬種值（選擇性高）
			  -- gender 只有 2 種值（選擇性低）
			  -- 應該建：(user_id, gender) 而不是 (gender, user_id)
			  ```
			- 常用查詢模式決定順序
			  ```sql
			  -- 如果 90% 的查詢是：WHERE city = X AND age = Y
			  -- 10% 的查詢是：WHERE age = Y
			  -- 應該建：(city, age) + 單獨的 (age) 索引
			  ```
- ## 交易隔離級別：
	- ### ACID 先說（基礎概念）
	  | 特性 | 說明 | 例子 |
	  | ---- | ---- | ---- |
	  | **A**tomicity 原子性 | 全部成功或全部失敗 | 轉帳：扣款和入帳必須同時成功 |
	  | **C**onsistency 一致性 | 資料保持邏輯正確 | 轉帳前後，總金額不變 |
	  | **I**solation 隔離性 | 並發交易互不影響 | 兩個人同時轉帳，不會互相干擾 |
	  | **D**urability 持久性 | 提交後永久保存 | 交易成功後，即使斷電也不會丟失 |
	- **為什麼需要隔離級別？**
		- **沒有隔離**：效能最好，但會有各種問題（髒讀、不可重複讀、幻讀）
		- **完全隔離**：最安全，但效能最差（所有交易排隊執行）
	- ### 四種隔離級別（從低到高）：
		- **Read Uncommitted** → 可以讀到其他交易「未提交」的資料
		  ```sql
		  -- Transaction A
		  BEGIN;
		  UPDATE account SET balance = 1000 WHERE id = 1;
		  -- 還沒 COMMIT
		  
		  -- Transaction B（同時進行）
		  BEGIN;
		  SELECT balance FROM account WHERE id = 1;
		  -- 讀到 1000（Transaction A 還沒提交！）
		  ```
		  問題：髒讀（Dirty Read）
		  ```sql
		  Transaction A: UPDATE balance = 1000
		  Transaction B: 讀到 1000
		  Transaction A: ROLLBACK（回滾了！）
		  Transaction B: 以為 balance 是 1000，但其實不是
		  ```
		- **Read Committed**   → **只能讀到已提交的資料***（PostgreSQL 預設 可以改成用其他）
		  ```sql
		  -- Transaction A
		  BEGIN;
		  UPDATE account SET balance = 1000 WHERE id = 1;
		  -- 還沒 COMMIT
		  
		  -- Transaction B
		  BEGIN;
		  SELECT balance FROM account WHERE id = 1;
		  -- 讀到舊值（例如 500）
		  
		  -- Transaction A
		  COMMIT;  -- 現在提交了
		  
		  -- Transaction B
		  SELECT balance FROM account WHERE id = 1;
		  -- 現在讀到 1000
		  ```
		  **解決了：髒讀** ✅
			- **問題：不可重複讀（Non-Repeatable Read）**
			  ```sql
			  Transaction B 在同一個交易中讀了兩次
			  第一次讀到 500
			  第二次讀到 1000（因為 A 提交了）
			  → 同一個交易內，兩次讀取結果不一樣！
			  ```
		- **Repeatable Read**  → 同一個交易內，多次讀取結果一致（MySQL 預設）
		  ```sql
		  -- Transaction B
		  BEGIN;
		  SELECT balance FROM account WHERE id = 1;  -- 讀到 500
		  
		  -- Transaction A（同時進行）
		  BEGIN;
		  UPDATE account SET balance = 1000 WHERE id = 1;
		  COMMIT;
		  
		  -- Transaction B
		  SELECT balance FROM account WHERE id = 1;  -- 還是讀到 500！
		  COMMIT;
		  ```
		  
		  **解決了：不可重複讀** ✅
		  
		  **如何實現？MVCC（Multi-Version Concurrency Control）**
		  ```
		  Transaction B 開始時，會記錄一個「版本號」
		  之後的讀取都讀這個版本的資料
		  即使其他交易提交了新版本，B 還是讀舊版本
		  ```
			- **問題：幻讀（Phantom Read）**
			  同一個交易中，兩次「範圍查詢」得到不同的結果集
			  不是單筆資料變了（那是不可重複讀），是**資料筆數變了**（有新增或刪除）
			  ```sql
			  -- Transaction A
			  BEGIN;
			  SELECT COUNT(*) FROM account WHERE balance > 100;  -- 結果：10 筆
			  
			  -- Transaction B（同時進行）
			  BEGIN;
			  INSERT INTO account (id, balance) VALUES (11, 200);
			  COMMIT;
			  
			  -- Transaction A
			  SELECT COUNT(*) FROM account WHERE balance > 100;  -- 結果：11 筆？
			  ```
			- 面試常問：「MySQL 的 Repeatable Read 是怎麼避免幻讀的？」
				- 答：MySQL 用 Next-Key Lock 避免
				  ```sql
				  -- Transaction A
				  BEGIN;
				  SELECT * FROM account WHERE id > 10 FOR UPDATE;
				  -- MySQL 會鎖定：
				  -- 1. 所有 id > 10 的現有資料（Record Lock）
				  -- 2. id > 10 的「間隙」（Gap Lock）
				  
				  -- Transaction B
				  INSERT INTO account (id, balance) VALUES (15, 200);
				  -- 被 Block！因為 15 在被鎖定的間隙內
				  ```
		- **Serializable** → 完全隔離，就像交易排隊執行一樣
		  ```sql
		  -- Transaction A
		  BEGIN;
		  SELECT * FROM account WHERE balance > 100;
		  
		  -- Transaction B（想插入資料）
		  BEGIN;
		  INSERT INTO account (id, balance) VALUES (11, 200);
		  -- 會被 Block，等 A 結束
		  ```
	- ### 實戰技巧
	- **如何選擇隔離級別？**
		- **一般 Web 應用**：Read Committed 足夠
			- 例如：部落格、論壇
		- **需要一致性讀取**：Repeatable Read
			- 例如：報表生成、資料分析
		- **金融交易**：Serializable
			- 例如：銀行轉帳、訂單支付
		- **高並發讀取**：Read Committed + 樂觀鎖
			- 例如：秒殺系統
- **正規化：**
	- 1NF：每個欄位不可再分
	- 2NF：消除部分依賴（非主鍵欄位完全依賴主鍵）
	- 3NF：消除傳遞依賴
	- 反正規化：為了效能犧牲一些正規化（常見於遊戲後端）
- **MySQL vs PostgreSQL：**
  | 特性 | MySQL | PostgreSQL |
  |------|-------|------------|
  | ACID | InnoDB 支援 | 完整支援 |
  | 複雜查詢 | 較弱 | 強大（支援 window function, CTE）|
  | JSON | 5.7+ 支援 | 原生支援且功能更強 |
  | 複製 | Master-Slave | Streaming Replication |
  | 預設隔離級別 | Repeatable Read | Read Committed |