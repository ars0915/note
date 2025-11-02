public:: true
tags:: golang, lock

- ## sync.Mutex
	- 互斥鎖，同時只能一個 goroutine 持有
	- Lock/Unlock 必須成對
- ## sync.RWMutex
	- 讀寫鎖
	- 多個 reader 可同時持有（RLock）
	- 只能一個 writer 持有（Lock）
	- Writer 會 block 所有 readers 和 writers
	- 適用場景：讀多寫少
	  ```go
	  var rwmu sync.RWMutex
	  
	  *// Reader*
	  rwmu.RLock()
	  *// read data*
	  rwmu.RUnlock()
	  
	  *// Writer*
	  rwmu.Lock()
	  *// write data*
	  rwmu.Unlock()
	  ```
	- ### 排序邏輯
		- 多個 Reader 並發：如果沒有 Writer 在等待，所有 Reader 都可以同時獲取鎖
		- Writer 獨佔：Writer 必須等所有 Reader 都釋放鎖後才能獲取
		- 防止 Writer Starvation：當有 Writer 在等待時，**新來的 Reader 會被阻塞**
		- 場景1：
		  ```go
		  R1.RLock() ✓ 獲取成功
		  R2.RLock() ✓ 獲取成功（與 R1 並發）
		  R3.RLock() ✓ 獲取成功（與 R1, R2 並發）
		  ```
		  → 三個 reader 同時持有鎖
		- 場景 2：Writer 來了
		  ````
		  時間軸：
		  1. R1.RLock() ✓ 獲取成功
		  2. R2.RLock() ✓ 獲取成功
		  3. W1.Lock()   ⏸ 等待（等 R1, R2 釋放）
		  4. R3.RLock()  ⏸ 阻塞（因為 W1 在等待）← 關鍵！
		  5. R1.RUnlock()
		  6. R2.RUnlock()
		  7. W1 獲取鎖 ✓
		  8. W1.Unlock()
		  9. R3 獲取鎖 ✓
		  ```
			- **關鍵點：** 步驟 4 的 R3 會被阻塞，即使此時還有其他 reader 持有鎖。這是為了避免 writer starvation（寫者饑餓）。
		- 場景 3：Writer 釋放後
		  ```
		  1. W1.Lock()  ✓ 獲取成功
		  2. R1.RLock() ⏸ 等待
		  3. R2.RLock() ⏸ 等待
		  4. R3.RLock() ⏸ 等待
		  5. W1.Unlock()
		  6. R1, R2, R3 同時獲取鎖 ✓ （並發執行）
		  ```
		- 為什麼要這樣設計？
			- **如果不阻塞新來的 Reader：**
			  ```go
			  假設 reader 源源不絕：
			  R1.RLock() ✓
			  R2.RLock() ✓
			  W1.Lock()  ⏸ 等待
			  R3.RLock() ✓ （如果允許）
			  R4.RLock() ✓ （如果允許）
			  R5.RLock() ✓ （如果允許）
			  ...
			  ```
			  → W1 永遠拿不到鎖！
- ## atomic operations
	- 不算是鎖，但可用於簡單的原子操作
	- 比 Mutex 更快
	- 只適用於簡單類型（int, pointer 等）
	  
	  ```go
	  var counter int64
	  atomic.AddInt64(&counter, 1)
	  value := atomic.LoadInt64(&counter)
	  ```
- ## sync.Map
	- Thread-safe 的 map
	- 內部已經處理好同步
	- 適用於讀多寫少的場景
- ## 選擇建議:
	- 簡單計數器 → atomic
	- 保護單一資源 → Mutex
	- 讀多寫少 → RWMutex
	- 優先考慮用 channel 來設計（"share memory by communicating"）
- ## 使用 channel
	- **Mutex 方式：** Share memory by communicating（共享記憶體）→ 需要加鎖保護
	  **Channel 方式：** Communicate by sharing memory（透過通信共享）→ 只有一個 goroutine 擁有資料
	- ## 實際對比例子
	- 例子 1：計數器
		- **❌ 用 Mutex（傳統方式）：**
		  ```go
		  type Counter struct {
		      mu    sync.Mutex
		      value int
		  }
		  
		  func (c *Counter) Increment() {
		      c.mu.Lock()
		      c.value++
		      c.mu.Unlock()
		  }
		  
		  func (c *Counter) Get() int {
		      c.mu.Lock()
		      defer c.mu.Unlock()
		      return c.value
		  }
		  
		  // 使用
		  counter := &Counter{}
		  go counter.Increment()
		  go counter.Increment()
		  fmt.Println(counter.Get())
		  ```
		- ✅ 用 Channel（Go 方式）：
		  ```go
		  type Counter struct {
		      requests chan func(int) int
		      value    int
		  }
		  
		  func NewCounter() *Counter {
		      c := &Counter{
		          requests: make(chan func(int) int),
		      }
		      go c.run() // 只有這個 goroutine 能修改 value
		      return c
		  }
		  
		  func (c *Counter) run() {
		      for fn := range c.requests {
		          c.value = fn(c.value)
		      }
		  }
		  
		  func (c *Counter) Increment() {
		      c.requests <- func(v int) int { return v + 1 }
		  }
		  
		  func (c *Counter) Get() int {
		      result := make(chan int)
		      c.requests <- func(v int) int {
		          result <- v
		          return v
		      }
		      return <-result
		  }
		  
		  // 使用
		  counter := NewCounter()
		  go counter.Increment()
		  go counter.Increment()
		  fmt.Println(counter.Get())
		  ```
		- **差異：**
			- Mutex：多個 goroutine 共享記憶體，用鎖保護
			- Channel：只有一個 goroutine 擁有資料，其他透過 channel 發送請求
	- 例子 2：Connection Pool（更實際）
		- ❌ 用 Mutex：
		  ```go
		  type Pool struct {
		      mu    sync.Mutex
		      conns []*Conn
		  }
		  
		  func (p *Pool) Get() *Conn {
		      p.mu.Lock()
		      defer p.mu.Unlock()
		      
		      if len(p.conns) == 0 {
		          return nil
		      }
		      conn := p.conns[0]
		      p.conns = p.conns[1:]
		      return conn
		  }
		  
		  func (p *Pool) Put(conn *Conn) {
		      p.mu.Lock()
		      defer p.mu.Unlock()
		      p.conns = append(p.conns, conn)
		  }
		  ```
		- ✅ 用 Channel：
		  ```go
		  type Pool struct {
		      conns chan *Conn
		  }
		  
		  func NewPool(size int) *Pool {
		      return &Pool{
		          conns: make(chan *Conn, size),
		      }
		  }
		  
		  func (p *Pool) Get() *Conn {
		      select {
		      case conn := <-p.conns:
		          return conn
		      default:
		          return nil
		      }
		  }
		  
		  func (p *Pool) Put(conn *Conn) {
		      select {
		      case p.conns <- conn:
		      default:
		          // pool 滿了，丟棄
		          conn.Close()
		      }
		  }
		  ```
		- **優勢：**
			- 更簡潔，不需要 defer unlock
			- Buffered channel 自帶容量限制
			- Select 可以處理 timeout
	- 例子 3：Worker Pool（你可能實際遇到）
	  場景：處理多個 streaming connection
		- ✅ Channel 方式（推薦）：
		  ```go
		  type Task struct {
		      ConnID int
		      Data   []byte
		  }
		  
		  func worker(id int, tasks <-chan Task, results chan<- Result) {
		      for task := range tasks {
		          // 處理任務
		          result := process(task)
		          results <- result
		      }
		  }
		  
		  func main() {
		      tasks := make(chan Task, 100)
		      results := make(chan Result, 100)
		      
		      // 啟動 10 個 worker
		      for i := 0; i < 10; i++ {
		          go worker(i, tasks, results)
		      }
		      
		      // 發送任務
		      go func() {
		          for _, task := range getTasks() {
		              tasks <- task
		          }
		          close(tasks) // 重要：關閉通知 workers 結束
		      }()
		      
		      // 收集結果
		      for i := 0; i < totalTasks; i++ {
		          result := <-results
		          handleResult(result)
		      }
		  }
		  ```
		- **為什麼這裡用 channel 更好：**
			- 自然的 producer-consumer 模式
			- 不需要管理 worker 的同步
			- close(tasks) 可以優雅地通知所有 worker 結束
	- 例子 4：State Machine（狀態機）
	  假設你在處理 streaming connection 的狀態：
		- ✅ Channel 方式：
		  ```go
		  type ConnState int
		  
		  const (
		      StateIdle ConnState = iota
		      StateConnecting
		      StateStreaming
		      StateClosed
		  )
		  
		  type StateEvent struct {
		      Event string
		      Reply chan error
		  }
		  
		  type Connection struct {
		      state  ConnState
		      events chan StateEvent
		  }
		  
		  func (c *Connection) run() {
		      for event := range c.events {
		          err := c.handleEvent(event.Event)
		          if event.Reply != nil {
		              event.Reply <- err
		          }
		      }
		  }
		  
		  func (c *Connection) handleEvent(event string) error {
		      switch c.state {
		      case StateIdle:
		          if event == "connect" {
		              c.state = StateConnecting
		              return c.doConnect()
		          }
		      case StateConnecting:
		          if event == "connected" {
		              c.state = StateStreaming
		              return nil
		          }
		      // ...
		      }
		      return fmt.Errorf("invalid transition")
		  }
		  
		  func (c *Connection) Connect() error {
		      reply := make(chan error)
		      c.events <- StateEvent{Event: "connect", Reply: reply}
		      return <-reply
		  }
		  ```
		- **優勢：**
			- 所有狀態變更都在一個 goroutine，不會有 race condition
			- 事件順序天然保證（channel 是 FIFO）
- ## 什麼時候用 Channel vs Mutex？
	- ### 用 Channel：
		- 所有權轉移（ownership transfer）
		  ```go
		  // 傳遞 connection 給 worker
		  workerCh <- conn
		  ```
		- 分發任務（distributing work）
		  ```go
		  // Worker pool
		  tasks := make(chan Task, 100)
		  ```
		- 聚合結果（aggregating results）
		  ```go
		  // 從多個 goroutine 收集結果
		  results := make(chan Result, numWorkers)
		  ```
		- 通知訊號（signaling）
		  ```go
		  // 通知 goroutine 該停止了
		  done := make(chan struct{})
		  close(done)
		  ```
	- ### 用 Mutex：
		- 簡單的共享狀態
		- Caching
		- 短暫的臨界區
	-