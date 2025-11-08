public:: true
tags:: golang

- ## sync.Pool
  臨時物件池，主要用來減少 GC 壓力和提升效能。
	- 特性
		- **臨時性**：Pool 裡的物件隨時可能被 GC 回收（不保證一定存在）
		- **併發安全**：多個 goroutine 可以安全地存取
		- **自動清理**：GC 時會清空 Pool（每兩次 GC 之間清一次）
	- ### 為什麼需要 sync.Pool
		- **減少記憶體分配次數**
		  ```go
		  // ❌ 不好的做法：頻繁分配記憶體
		  func handleRequest() {
		      buf := make([]byte, 64*1024)  // 每次都分配 64KB
		      // ... 使用 buf
		      // buf 用完就被丟棄 → GC 壓力大
		  }
		  
		  // ✅ 好的做法：重複使用物件
		  var bufferPool = sync.Pool{
		      New: func() interface{} {
		          return make([]byte, 64*1024)
		      },
		  }
		  
		  func handleRequest() {
		      buf := bufferPool.Get().([]byte)
		      defer bufferPool.Put(buf)  // 用完放回
		      // ... 使用 buf
		  }
		  ```
		- **降低 GC 壓力**
		- **提升效能（特別是高併發場景）**
	- 常用場景
		- Buffer Pool（最常見）
		- 格式化工具（fmt.Sprintf 內部用法）
		- 連線池相關（Redis, DB）
	- 常見陷阱
		- 忘記 Reset
		  ```go
		  // ❌ 錯誤：不 reset
		  buf := bufferPool.Get().(*bytes.Buffer)
		  buf.WriteString("old data")
		  bufferPool.Put(buf)
		  
		  buf2 := bufferPool.Get().(*bytes.Buffer)
		  // buf2 可能包含 "old data"！
		  
		  // ✅ 正確：使用前 reset
		  buf := bufferPool.Get().(*bytes.Buffer)
		  buf.Reset()  // 清空舊資料
		  buf.WriteString("new data")
		  ```
		- 把 Pool 當永久儲存 **sync.Pool 在 GC 發生時會被清空**
		  ```go
		  // ❌ 錯誤：期待物件永遠存在
		  pool.Put(importantData)
		  // GC 發生...
		  data := pool.Get()  // 可能是 nil！
		  ```
		- Pool 裡存了帶狀態的物件
-