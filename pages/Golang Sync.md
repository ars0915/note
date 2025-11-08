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
- ## sync.Once
  保證某個函數在併發環境下只執行一次
	- **執行一次**：無論多少個 goroutine 同時呼叫，函數只會被執行一次
	- **併發安全**：內部使用 atomic + mutex 實現
	- **阻塞等待**：第一個 goroutine 執行時，其他 goroutine 會等待它完成
- ## sync.Cond
  條件變數（Condition Variable），用於 goroutine 之間的等待/通知機制
	- 特性
		- **等待條件**：goroutine 可以等待某個條件成立
		- **通知機制**：其他 goroutine 可以通知等待者條件已成立
		- **必須配合鎖**：所有操作必須在持有 Mutex 的情況下進行
	- 為什麼需要 sync.Cond？
		- 不好的做法：忙等待（Busy-waiting）
		  ```go
		  var (
		      mu    sync.Mutex
		      ready bool
		  )
		  
		  // ❌ 浪費 CPU
		  func waitForReady() {
		      for {
		          mu.Lock()
		          if ready {
		              mu.Unlock()
		              break
		          }
		          mu.Unlock()
		          time.Sleep(10 * time.Millisecond)  // 輪詢，浪費 CPU
		      }
		      fmt.Println("條件成立，繼續執行")
		  }
		  ```
		- 使用 sync.Cond：高效等待
		  ```go
		  var (
		      mu    sync.Mutex
		      cond  = sync.NewCond(&mu)
		      ready bool
		  )
		  
		  // ✅ 高效：阻塞等待，不浪費 CPU
		  func waitForReady() {
		      mu.Lock()
		      for !ready {  // 循環檢查條件
		          cond.Wait()  // 釋放鎖並阻塞，等待被喚醒
		      }
		      mu.Unlock()
		      fmt.Println("條件成立，繼續執行")
		  }
		  
		  func setReady() {
		      mu.Lock()
		      ready = true
		      cond.Signal()  // 喚醒一個等待的 goroutine
		      mu.Unlock()
		  }
		  ```
	- 三個核心方法
		- Wait() - 等待
		  ```go
		  cond.Wait()
		  ```
			- **做了三件事：**
				- 1. 自動釋放鎖（unlock）
				- 2. 阻塞等待（block）
				- 3. 被喚醒後，重新獲得鎖（lock）
			- **執行流程：**
			  ```
			  goroutine 持有鎖
			      ↓
			  呼叫 cond.Wait()
			      ↓
			  自動釋放鎖（其他 goroutine 可以進來）
			      ↓
			  阻塞等待...
			      ↓
			  被 Signal/Broadcast 喚醒
			      ↓
			  自動重新獲得鎖
			      ↓
			  從 Wait() 返回
			  ```
			- **為什麼 Wait() 必須在 for 循環中使用？**
				- 因為可能發生虛假喚醒（spurious wakeup）。Wait() 被喚醒時，條件不一定成立，可能是因為：
					- 多個 goroutine 競爭，條件被其他人改變了
					- 系統層面的虛假喚醒
				- 使用 for 循環可以在喚醒後重新檢查條件，如果條件還不成立，就繼續 Wait()。這是條件變數的標準用法，不只 Go，其他語言（C++、Java）也是一樣。"
		- Signal() - 喚醒一個
			- 喚醒**一個**等待的 goroutine
			- 如果沒有等待者，什麼都不做
			- **必須在持有鎖的情況下呼叫**
		- Broadcast() - 喚醒所有
			- 喚醒**所有**等待的 goroutine
			- 如果沒有等待者，什麼都不做
			- **必須在持有鎖的情況下呼叫**
	- 常用場景
		- 生產者-消費者模型
		  ```go
		  type Queue struct {
		      mu    sync.Mutex
		      cond  *sync.Cond
		      queue []int
		  }
		  
		  func NewQueue() *Queue {
		      q := &Queue{
		          queue: make([]int, 0),
		      }
		      q.cond = sync.NewCond(&q.mu)
		      return q
		  }
		  
		  // 生產者：加入元素
		  func (q *Queue) Enqueue(item int) {
		      q.mu.Lock()
		      defer q.mu.Unlock()
		      
		      q.queue = append(q.queue, item)
		      fmt.Printf("生產: %d (queue size: %d)\n", item, len(q.queue))
		      
		      q.cond.Signal()  // 通知一個消費者
		  }
		  
		  // 消費者：取出元素
		  func (q *Queue) Dequeue() int {
		      q.mu.Lock()
		      defer q.mu.Unlock()
		      
		      // ⚠️ 必須用 for 循環
		      for len(q.queue) == 0 {
		          fmt.Println("消費者等待...")
		          q.cond.Wait()  // 隊列為空，等待
		      }
		      
		      item := q.queue[0]
		      q.queue = q.queue[1:]
		      fmt.Printf("消費: %d (queue size: %d)\n", item, len(q.queue))
		      
		      return item
		  }
		  
		  // 使用範例
		  func main() {
		      q := NewQueue()
		      
		      // 啟動 3 個消費者
		      for i := 1; i <= 3; i++ {
		          go func(id int) {
		              for j := 0; j < 3; j++ {
		                  item := q.Dequeue()
		                  fmt.Printf("消費者 %d 得到: %d\n", id, item)
		                  time.Sleep(500 * time.Millisecond)
		              }
		          }(i)
		      }
		      
		      // 生產者：每 200ms 生產一個
		      time.Sleep(1 * time.Second)
		      for i := 1; i <= 9; i++ {
		          q.Enqueue(i)
		          time.Sleep(200 * time.Millisecond)
		      }
		      
		      time.Sleep(3 * time.Second)
		  }
		  ```
		- 等待所有 goroutine 準備好
		  ```go
		  type Barrier struct {
		      mu      sync.Mutex
		      cond    *sync.Cond
		      count   int      // 當前等待數量
		      target  int      // 目標數量
		      round   int      // 輪次（防止重複使用）
		  }
		  
		  func NewBarrier(n int) *Barrier {
		      b := &Barrier{
		          target: n,
		      }
		      b.cond = sync.NewCond(&b.mu)
		      return b
		  }
		  
		  // 等待所有人到齊
		  func (b *Barrier) Wait() {
		      b.mu.Lock()
		      defer b.mu.Unlock()
		      
		      b.count++
		      currentRound := b.round
		      
		      if b.count < b.target {
		          // 還沒到齊，等待
		          for b.round == currentRound {
		              b.cond.Wait()
		          }
		      } else {
		          // 到齊了，喚醒所有人
		          b.count = 0
		          b.round++
		          b.cond.Broadcast()
		      }
		  }
		  
		  // 使用範例：賽馬比賽
		  func main() {
		      barrier := NewBarrier(3)
		      
		      for i := 1; i <= 3; i++ {
		          go func(id int) {
		              fmt.Printf("賽馬 %d: 準備中...\n", id)
		              time.Sleep(time.Duration(id*200) * time.Millisecond)
		              
		              fmt.Printf("賽馬 %d: 就位！\n", id)
		              barrier.Wait()  // 等待所有馬就位
		              
		              fmt.Printf("賽馬 %d: 出發！🏇\n", id)
		          }(i)
		      }
		      
		      time.Sleep(3 * time.Second)
		  }
		  
		  // 輸出：
		  // 賽馬 1: 準備中...
		  // 賽馬 2: 準備中...
		  // 賽馬 3: 準備中...
		  // 賽馬 1: 就位！
		  // 賽馬 2: 就位！
		  // 賽馬 3: 就位！
		  // 賽馬 3: 出發！🏇
		  // 賽馬 1: 出發！🏇
		  // 賽馬 2: 出發！🏇
		  ```
		- 讀寫分離的緩存更新通知
		  ```go
		  type Cache struct {
		      mu      sync.RWMutex
		      cond    *sync.Cond
		      data    map[string]string
		      version int
		  }
		  
		  func NewCache() *Cache {
		      c := &Cache{
		          data: make(map[string]string),
		      }
		      c.cond = sync.NewCond(c.mu.RLocker())  // ⚠️ 注意：用 RLocker
		      return c
		  }
		  
		  // 寫入數據並通知
		  func (c *Cache) Set(key, value string) {
		      c.mu.Lock()
		      defer c.mu.Unlock()
		      
		      c.data[key] = value
		      c.version++
		      
		      c.cond.Broadcast()  // 通知所有等待者
		  }
		  
		  // 等待特定版本
		  func (c *Cache) WaitForVersion(targetVersion int) {
		      c.mu.RLock()
		      defer c.mu.RUnlock()
		      
		      for c.version < targetVersion {
		          c.cond.Wait()
		      }
		  }
		  ```
	- **Channel vs Cond**
		- | 場景 | 推薦 | 原因 |
		  | ---- | ---- | ---- |
		  | 傳遞數據 | Channel | Channel 天生就是為此設計 |
		  | 單純的通知（無數據） | Channel | 更簡單直觀 |
		  | 需要 Broadcast 多個等待者 | Cond | Channel 只能一對一 |
		  | 與現有鎖配合使用 | Cond | 不用引入額外的同步機制 |
		  | 複雜的條件等待 | Cond | 可以在鎖內檢查複雜條件 |
	- **WaitGroup vs Cond**
		- | 特性 | WaitGroup | Cond |
		  | ---- | ---- | ---- |
		  | 用途 | 等待固定數量任務 | 等待條件成立 |
		  | 條件 | 簡單計數 | 任意複雜條件 |
		  | 重複使用 | 可以重置 | 可以重複等待 |
		  | 通知方式 | Done() | Signal/Broadcast |
	-