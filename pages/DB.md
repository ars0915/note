- **Index 原理與優化：**
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
		- 使用 `!=`、`<>`、`OR`
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
- 交易隔離級別：
	- Read Uncommitted → 可能髒讀
	- Read Committed   → 可能不可重複讀（PostgreSQL 預設）
	- Repeatable Read  → 可能幻讀（MySQL 預設）
	- Serializable     → 效能最差但最安全
- 面試常問：「MySQL 的 Repeatable Read 是怎麼避免幻讀的？」
	- 答：MVCC + Gap Lock
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