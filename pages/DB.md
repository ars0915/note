- **Index 原理與優化：**
	- B+ Tree 為什麼適合做索引？
		- 葉子節點有指針連接，適合範圍查詢
		- 非葉子節點只存 key，能存更多索引
		- 高度低（通常 3-4 層），IO 次數少
	- 什麼時候索引會失效？
		- 使用 `!=`、`<>`、`OR`
		- Like 開頭用 `%`
		- 對索引欄位做函數運算
		- 隱式類型轉換
	- 聯合索引的最左匹配原則
		- `(a, b, c)` 的索引只能用於 `a`、`a,b`、`a,b,c` 的查詢
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