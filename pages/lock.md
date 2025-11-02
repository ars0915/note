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
- # Golang CAS 自旋
	- ### CAS (Compare-And-Swap)
		- 原子操作：比較記憶體值，如果相等就交換。
		  ```go
		  *// 偽代碼*
		  func CompareAndSwap(addr *int32, old, new int32) bool {
		    if *addr == old {
		        *addr = new
		        return true  *// 成功*
		    }
		    return false     *// 失敗（值已被其他人改過）*
		  }
		  ```
	- ### 自旋 (Spin)
		- 在 **迴圈** 中不斷重試，直到成功為止。
		  ```go
		  for {
		    if CAS成功 {
		        break  *// 跳出迴圈*
		    }
		    *// 失敗就繼續重試（自旋）*
		  }
		  ```
	- ### CAS 自旋 = CAS + 迴圈重試
	  結合起來就是：用 CAS 操作，失敗就在迴圈中重試。
- ## Mutex 也用了 CAS
	- `sync.Mutex` 的 `Lock()` 實現：
	  ```go
	  func (m *Mutex) Lock() {
	      // 快速路徑：嘗試用 CAS 獲取鎖
	      if atomic.CompareAndSwapInt32(&m.state, 0, mutexLocked) {
	          return  // 成功就直接返回
	      }
	      // 失敗才進入慢速路徑
	      m.lockSlow()
	  }
	  ```
	- 純 CAS 自旋非常慢，因為浪費 CPU 在重試
-
-
- # Reference
	- omegaatt, "golang 樂觀鎖、悲觀鎖 學習筆記與實驗," **, Available: [link_to_page](https://www.omegaatt.com/blogs/develop/2023/golang_lock_practice/). 
	  type:: [[Web Page]]