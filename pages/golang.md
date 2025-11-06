public:: true

- ## goroutine GMP
	- **基本概念要能說出來：**
		- **G (Goroutine)**：用戶態的輕量級執行緒
		- **M (Machine)**：OS 執行緒
		- **P (Processor)**：邏輯處理器，持有 G 的佇列
	- **核心優勢**：M:N 調度模型，一個 M 可以執行多個 G
- ## Goroutine Leak 怎麼避免？
	- channel 沒關閉
	- context 沒取消
	- 死鎖情況
- ## Channel 的使用與原理
  id:: 8ca97253-f228-4ce0-8605-ca9e51f24c3d
	- buffered vs unbuffered
	- close 的時機
	- select 的使用
	- ### 超時控制
		- time.After
		  ```go
		  func doBadthing(done chan bool) {
		  	time.Sleep(time.Second)
		  	done <- true
		  }
		  
		  // 或是使用 select 尝试向信道 done 发送信号，如果发送失败，则说明缺少接收者(receiver)，即超时了，那么直接退出即可。
		  func doGoodthing(done chan bool) {
		  	time.Sleep(time.Second)
		  	select {
		  	case done <- true:
		  	default:
		  		return
		  	}
		  }
		  
		  func timeout(f func(chan bool)) error {
		  	done := make(chan bool, 1) // 创建channel done 时，缓冲区设置为 1，即使没有接收方，发送方也不会发生阻塞。
		  	go f(done)
		  	select {
		  	case <-done:
		  		fmt.Println("done")
		  		return nil
		  	case <-time.After(time.Millisecond):
		  		return fmt.Errorf("timeout")
		  	}
		  }
		  
		  // timeout(doBadthing)
		  ```
		  如果 done 是一个无缓冲区的 channel，如果没有超时，`doBadthing` 中会向 done 发送信号，`select` 中会接收 done 的信号，因此 `doBadthing` 能够正常退出，子协程也能够正常退出。
		  但是，当超时发生时，select 接收到 `time.After` 的超时信号就返回了，`done` 没有了接收方(receiver)，而 `doBadthing` 在执行 1s 后向 `done` 发送信号，由于没有接收者且无缓存区，发送者(sender)会一直阻塞，导致协程不能退出。
	- ### 優雅關閉
		- ### channel 的三种状态和三种操作结果
		  
		  | 操作 | 空值(nil) | 非空已关闭 | 非空未关闭 |
		  | ---- | ---- | ---- |
		  | 关闭 | panic | panic | 成功关闭 |
		  | 发送数据 | 永久阻塞 | panic | 阻塞或成功发送 |
		  | 接收数据 | 永久阻塞 | 永不阻塞 | 阻塞或者成功接收 |
		- ```go
		  func doCheckClose(taskCh chan int) {
		  	for {
		  		select {
		  		case t, beforeClosed := <-taskCh:
		  			if !beforeClosed {
		  				// beforeClosed 为 false 表示信道已被关闭。若关闭，则不再阻塞等待，直接返回，对应的协程随之退出。
		  				fmt.Println("taskCh has been closed")
		  				return
		  			}
		  			time.Sleep(time.Millisecond)
		  			fmt.Printf("task %d is done\n", t)
		  		}
		  	}
		  }
		  
		  func sendTasksCheckClose() {
		  	taskCh := make(chan int, 10)
		  	go doCheckClose(taskCh)
		  	for i := 0; i < 1000; i++ {
		  		taskCh <- i
		  	}
		  	// sendTasks 函数中，任务发送结束之后，使用 close(taskCh) 将 channel taskCh 关闭。
		  	close(taskCh)
		  }
		  
		  func TestDoCheckClose(t *testing.T) {
		  	t.Log(runtime.NumGoroutine())
		  	sendTasksCheckClose()
		  	time.Sleep(time.Second)
		  	runtime.GC()
		  	t.Log(runtime.NumGoroutine())
		  }
		  ```
	- ### 通道关闭原则
		- 如果 channel 已经被关闭，再次关闭会产生 panic，这时通过 recover 使程序恢复正常。
		- 礼貌的方式
			- 使用 sync.Once 或互斥锁(sync.Mutex)确保 channel 只被关闭一次。
		- 优雅的方式
			- 情形一：M个接收者和一个发送者，发送者通过关闭用来传输数据的通道来传递发送结束信号。
			- 情形二：一个接收者和N个发送者，此唯一接收者通过关闭一个额外的信号通道来通知发送者不要再发送数据了。
			- 情形三：M个接收者和N个发送者，它们中的任何协程都可以让一个中间调解协程帮忙发出停止数据传送的信号。
- ## sync 包的使用
	- WaitGroup、Mutex、RWMutex
	- Once、Pool
	- 什麼時候用 channel，什麼時候用 mutex？
- ## Context 的傳遞與取消
  Context 是 Go 用來在 goroutine 之間傳遞**取消信號、超時控制、截止時間和請求範圍值**的標準方式。
  | 功能 | 使用場景 | 函數 |
  | ---- | ---- | ---- |
  | **手動取消** | 用戶取消操作、錯誤時提前終止 | WithCancel |
  | **超時控制** | API 調用、資料庫查詢 | WithTimeout |
  | **截止時間** | 需要在特定時間點完成 | WithDeadline |
  | **優雅關閉** | 服務器關閉、批處理完成 | 配合 signal.Notify |
- ## 常見並發問題
	- Race condition 怎麼 debug？
	- 如何控制 goroutine 數量？（worker pool）
- ## panic
	- `recover()` **必須**在 `defer` 函數中呼叫
	  ```go
	  func wrong2() {
	      defer recover() // ❌ 無效！recover 必須在「函數呼叫」中
	      panic("test")
	  }
	  
	  // 正確寫法
	  func correct() {
	      defer func() {
	          recover() // ✅ 在匿名函數中呼叫
	      }()
	      panic("test")
	  }
	  ```
	- panic 發生後，當前函數會立即停止，開始執行 defer 棧
	- recover 要跟 panic 在同一個 goroutine 才會被觸發
	- 如果沒有 recover，panic 會向上傳播直到程式崩潰
	- defer 是 **LIFO（後進先出）** 棧
	- 適合用 panic 的場景：
		- **初始化階段的致命錯誤**（如設定檔載入失敗）
		- **程式設計錯誤**（如除以零、陣列越界）
		- **不可恢復的錯誤**（如記憶體分配失敗）
	- 不應該用 panic 的場景：
		- **正常的錯誤處理**（應該用 `error` 回傳）
		- **可預期的失敗**（如網路請求失敗、檔案不存在）
		- **業務邏輯錯誤**
	- **面試標準答案：**
		- "Go 提倡用 error 處理錯誤，panic 應該只用在真正無法恢復的異常情況。在函式庫開發中，幾乎不應該讓 panic 傳播給呼叫者，應該 recover 後轉成 error 回傳。"
- ## channel
	- Unbuffered Channel
		- 在每次要傳送或接收前都必需進行等待
		  對於一個沒有 buffer 的 channel ，channel 上一次只能有一個值，前一個傳完，才能傳下一個，不能允許一次上傳超過一個
	- buffered Channel
		- 如果是有 buffer 的 channel 那麼就 **不必在意對方是否接收到** ，只要在以是否仍有足夠的緩充空間，如果沒有空間了才會進行等待。
	- 接收不確定個數的 channel
		- 因此可以利用 `for + range` 來接收 channel 上的值，但是要注意一點，傳送最後一個值後必需利用 `close()` 將 channel 關閉，golang 才知道這是最後一個值
- ## array
	- array 的長度是固定的，兩個長度不同的 array 不能互相賦值
	- C 語言的 array 是指向第一個元素的指針， Go 不是。**Go 的 array 被傳遞時是複製一個。**
	  ```go
	  a := [...]int{1, 2, 3}
	  b := a
	  a[0] = 100
	  fmt.Println(a, b) // [100 2 3] [1 2 3]
	  ```
- ## slice
	- ### struct
	  ```go
	  struct {
	      ptr *[]T
	      len int
	      cap int
	  }
	  ```
	- 新增元素到 slice 時, 如果超過了 cap 的大小, 會分配內存來增大。當小於 2048 時是以 2 的倍數新增。
	- ### slice 操作不複製元素
	  ![image.png](../assets/image_1762429535751_0.png)
	  ```go
	  nums := make([]int, 0, 8)
	  nums = append(nums, 1, 2, 3, 4, 5)
	  nums2 := nums[2:4]
	  printLenCap(nums)  // len: 5, cap: 8 [1 2 3 4 5]
	  printLenCap(nums2) // len: 2, cap: 6 [3 4]
	  
	  nums2 = append(nums2, 50, 60)
	  printLenCap(nums)  // len: 5, cap: 8 [1 2 3 4 50]
	  printLenCap(nums2) // len: 4, cap: 6 [3 4 50 60]
	  ```
		- nums2 把 nums 拿來切片，底層指的是同一個 array
		- nums2 新增 50, 60 將底層 [4] 的位置改成 50，[5] 改成 60
		- 因為 nums 和 nums2 是指向同一個 array 所以也被改了值
	- ### slice 操作
		- **Copy**
		  ![image.png](../assets/image_1762429852485_0.png)
		- **append**
		  ![image.png](../assets/image_1762429867927_0.png)
		  切片有三个属性，指针(ptr)、长度(len) 和容量(cap)。append 时有两种场景：
			- 当 append 之后的长度小于等于 cap，将会直接利用原底层数组剩余的空间。
			- 当 append 后的长度大于 cap 时，则会分配一块更大的区域来容纳新的底层数组。
			  因此，为了避免内存发生拷贝，如果能够知道最终的切片的大小，预先设置 cap 的值能够获得最好的性能。
		- **Delete**
		  ![image.png](../assets/image_1762429942293_0.png)
		  切片的底层是数组，因此删除意味着后面的元素需要逐个向前移位。每次删除的复杂度为 O(N)，因此切片不合适大量随机删除的场景，这种场景下适合使用链表。
		- **Delete(GC)**
		  ![image.png](../assets/image_1762429973856_0.png)
		  删除后，将空余的位置置空，有助于垃圾回收。
		- **Insert**
		  ![image.png](../assets/image_1762430007707_0.png)
		  insert 和 append 类似。即在某个位置添加一个元素后，将该位置后面的元素再 append 回去。复杂度为 O(N)。因此，不适合大量随机插入的场景。
		- **Filter**
		  ```go
		  n := 0
		  for _, x := range a {
		      if keep(x) {
		          a[n] = x  // 把保留的元素移到前面
		          n++       // n 記錄保留了幾個元素
		      }
		  }
		  a = a[:n]  // 重新切片，只保留前 n 個元素
		  
		  
		  // 假設原始 slice：a = [1, 2, 3, 4, 5]，我們要保留偶數。
		  // 循環後：
		  // a = [2, 4, 3, 4, 5]  ← 注意！長度還是 5
		  // n = 2                ← 只有 2 個元素符合條件
		  //      ↑  ↑  ←── 這些是舊數據
		  
		  // 執行 a = a[:n]
		  a = [2, 4]  // len: 2, 舊的 [3, 4, 5] 被切掉了
		  ```
		  当原切片不会再被使用时，就地 filter 方式是比较推荐的，可以节省内存空间。
			- `a = a[:n]` 的作用：
				- **截斷 slice**，讓長度變成實際保留的元素數量
				- **丟棄後面的舊元素**，讓 slice 只包含過濾後的數據
				- 這是一種**零分配（zero allocation）** 的高效過濾方式，不需要創建新 slice。
		- **Push**
		  ```go
		  a = append(a, x)
		  ```
		  在末尾追加元素，不考虑内存拷贝的情况，复杂度为 O(1)。
		  ![image.png](../assets/image_1762430302016_0.png)
		  在头部追加元素，时间和空间复杂度均为 O(N)，不推荐。
		- **Pop**
		  ![image.png](../assets/image_1762430334902_0.png)
		  尾部删除元素，复杂度 O(1)
		  ![image.png](../assets/image_1762430372145_0.png)
		  头部删除元素，如果使用切片方式，复杂度为 O(1)。但是需要注意的是，底层数组没有发生改变，第 0 个位置的内存仍旧没有释放。如果有大量这样的操作，头部的内存会一直被占用。
	- ### 性能陷阱 大量内存得不到释放
		- 在已有切片的基础上进行切片，不会创建新的底层数组。因为原来的底层数组没有发生变化，内存会一直占用，直到没有变量引用该数组。因此很可能出现这么一种情况，原切片由大量的元素构成，但是我们在原切片的基础上切片，虽然只使用了很小一段，但底层数组在内存中仍然占据了大量空间，得不到释放。比较推荐的做法，使用 `copy` 替代 `re-slice`。
		- ```go
		  func lastNumsBySlice(origin []int) []int {
		  	return origin[len(origin)-2:]
		  }
		  
		  func lastNumsByCopy(origin []int) []int {
		  	result := make([]int, 2)
		  	copy(result, origin[len(origin)-2:])
		  	return result
		  }
		  ```
		  `lastNumsBySlice` 耗费了 100.14 MB 内存，也就是说，申请的 100 个 1 MB 大小的内存没有被回收。因为切片虽然只使用了最后 2 个元素，但是因为与原来 1M 的切片引用了相同的底层数组，底层数组得不到释放，因此，最终 100 MB 的内存始终得不到释放。而 `lastNumsByCopy` 仅消耗了 3.14 MB 的内存。这是因为，通过 `copy`，指向了一个新的底层数组，当 origin 不再被引用后，内存会被垃圾回收(garbage collector, GC)。
- ## Go 進階陷阱
	- ### string/[]byte 轉換
		- **string 和 []byte 互轉會發生記憶體拷貝**，在高性能場景下可能成為瓶頸
		  ```go
		  s := "hello"
		  b := []byte(s)  // ⚠️ 分配新記憶體，拷貝 5 個字節
		  s2 := string(b) // ⚠️ 又分配新記憶體，再拷貝 5 個字節
		  ```
		  因為 string 是不可變的，[]byte 是可變的
		- 零拷貝技術
			- 使用 `unsafe` 包（不安全但高效）
			- 標準庫的零拷貝方案
				- `strings.Builder`（推薦）
				  ```go
				  func concatBuilder(strs []string) string {
				      var builder strings.Builder
				      for _, s := range strs {
				          builder.WriteString(s)  // 只在容量不足時才擴容
				      }
				      return builder.String()  // 最後一次轉換
				  }
				  ```
				- `bytes.Buffer`
				  ```go
				  var buf bytes.Buffer
				  buf.WriteString("hello")
				  buf.WriteByte(' ')
				  buf.WriteString("world")
				  result := buf.String()  // 最後才轉成 string
				  ```
	- ### for-range 閉包陷阱
		- 經典陷阱（Go 1.21 及之前）
		  ```go
		  func main() {
		      nums := []int{1, 2, 3}
		      var funcs []func()
		      
		      for _, n := range nums {
		          funcs = append(funcs, func() {
		              fmt.Println(n)  // ⚠️ n 是同一個變量！
		          })
		      }
		      
		      for _, f := range funcs {
		          f()
		      }
		  }
		  ```
		  
		  **輸出（Go 1.21）：**
		  ```
		  3
		  3
		  3  ← 全部都是 3！
		  ```
		- Go 1.22 起，for-range 循環變量在每次迭代都是新的！
		  ```go
		  // Go 1.22+
		  func main() {
		      nums := []int{1, 2, 3}
		      var funcs []func()
		      
		      for _, n := range nums {
		          funcs = append(funcs, func() {
		              fmt.Println(n)  // ✅ 現在正常了！
		          })
		      }
		      
		      for _, f := range funcs {
		          f()
		      }
		  }
		  ```
		  
		  **輸出（Go 1.22+）：**
		  ```
		  1
		  2
		  3  ← 符合預期！
		  ```
			- goroutine 中的陷阱
			  ```go
			  // ❌ 即使在 Go 1.22，這樣也有問題
			  for _, url := range urls {
			      go func() {
			          fetch(url)  // url 是外部變量
			      }()
			  }
			  
			  // ✅ 正確做法：作為參數傳入
			  for _, url := range urls {
			      go func(u string) {
			          fetch(u)
			      }(url)
			  }
			  ```
			- 指針陷阱
			  ```go
			  type User struct {
			      Name string
			  }
			  
			  users := []User{{"Alice"}, {"Bob"}}
			  var userPtrs []*User
			  
			  // ❌ 即使 Go 1.22 也有問題
			  for _, u := range users {
			      userPtrs = append(userPtrs, &u)  // u 的地址在每次迭代會變
			  }
			  
			  // 結果：可能都指向最後一個元素
			  
			  // ✅ 正確做法
			  for i := range users {
			      userPtrs = append(userPtrs, &users[i])
			  }
			  ```
		- 檢測工具
			- 使用  `go vet`
			  ```
			  *# 會檢測常見的閉包陷阱*
			  go vet ./...
			  ```
			- 使用靜態分析工具
			  ```
			  go install github.com/nishanths/exhaustive/cmd/exhaustive@latest
			  exhaustive ./...
			  ```
- ## ErrGroup
  `errgroup` 是 Go 官方擴展庫提供的工具，用於**管理一組 goroutine 的生命週期和錯誤處理**。
  
  ```go
  import "golang.org/x/sync/errgroup"
  ```
	- **核心功能：**
	- ✅ 等待一組 goroutine 完成（類似 `sync.WaitGroup`）
	- ✅ 收集第一個錯誤並取消其他 goroutine
- ✅ 自動管理 Context 取消
- ## TODO:
	- https://geektutu.com/post/hpg-sync-cond.html
	- ## 我的建議：針對面試準備
	- ### ✅  **必看** （直接影響面試表現）
	  
	  **第三章：並發編程**
	- ✅ sync.Pool（**必看**，面試常考，Redis 連線池等場景）
	- ✅ sync.Once（**必看**，單例模式，初始化場景）
	- ⚠️ sync.Cond（**選看**，面試較少考，但看一次搞懂概念就好）
	- ✅ 並發控制（**必看**，WaitGroup、Context、ErrGroup）