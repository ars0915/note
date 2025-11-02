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
	  ```
	  func BuyProductPessimistic(db *sql.DB, productID int, quantity int) error {
	      // 開始交易
	      tx, err := db.Begin()
	      if err != nil {
	          return err
	      }
	      defer tx.Rollback()
	      
	      // 步驟 1: 鎖定資料（其他人必須等待）
	      var product Product
	      err = tx.QueryRow(`
	          SELECT id, name, stock 
	          FROM products 
	          WHERE id = ? 
	          FOR UPDATE`,  -- ← 關鍵：加鎖
	          productID,
	      ).Scan(&product.ID, &product.Name, &product.Stock)
	      
	      if err != nil {
	          return err
	      }
	      
	      // 檢查庫存
	      if product.Stock < quantity {
	          return errors.New("insufficient stock")
	      }
	      
	      // 步驟 2: 更新（不需要檢查 version）
	      _, err = tx.Exec(`
	          UPDATE products 
	          SET stock = stock - ?
	          WHERE id = ?`,
	          quantity,
	          productID,
	      )
	      
	      if err != nil {
	          return err
	      }
	      
	      // 提交交易（釋放鎖）
	      return tx.Commit()
	  }
	  ```
	- 執行流程
	  ```
	  時間軸：
	  
	  用戶 A                              用戶 B
	  1. BEGIN
	  2. SELECT ... FOR UPDATE
	   ✓ 獲得鎖
	                                   3. BEGIN
	                                   4. SELECT ... FOR UPDATE
	                                      ⏸ 阻塞等待...
	  5. UPDATE stock
	  6. COMMIT
	   ✓ 釋放鎖
	                                      ✓ 獲得鎖
	                                   7. UPDATE stock
	                                   8. COMMIT
	  ```
	  **關鍵：** 用戶 B 必須等待用戶 A 完成（或超時）。
-