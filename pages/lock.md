public:: true

- # 樂觀鎖(Optimistic Locking)
	- 假設衝突很少發生，所以不先加鎖，而是在更新時檢查資料是否被修改過。
	- 典型實現：Version 欄位
		- **關鍵：** WHERE 條件中的 `version = ?` 確保只在資料沒被改過時才更新。
	- 其他實現方式
	  除了 version，還可以用：
		- Timestamp（時間戳）
		- Hash/Checksum（雜湊值）
		- 比較所有欄位（CAS - Compare And Swap）
- # 悲觀鎖 (Pessimistic Locking)
	- 假設衝突經常發生，所以先加鎖，確保獨佔資源。
	- 典型實現：SELECT FOR UPDATE