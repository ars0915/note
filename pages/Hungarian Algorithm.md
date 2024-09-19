public:: true
tags:: Algorithm, Graph

- ## 題目
	- func Solution(A []int, B []int, S int) bool{}
	  有N個病人，醫生有S個可能的appointment slots，每個病人有2 preferences，病人K想要看醫生在A[K] or B[K],醫生一次只能看一個病人
	  求是否可能看到所有病人？
	  i.e. A=[1,1,3] B=[2,2,1] S = 3 return true
	  因為醫生可以看 [1,2,3] or [2,1,3]
	  如果是 [2,2,1]就錯了
- ## Problem Breakdown
	- **病人**：每個病人有兩個可能的約會時間。
	  **醫生**：有 S 個約會時段可供選擇。
	  **目標**：檢查是否可以為每個病人分配一個約會時段，以滿足所有病人的需求，且每個醫生一次只能接受一個病人。
- ## Approach
	- **建模為二分圖**：
	  **左側節點**：病人。
	  **右側節點**：醫生的約會時段。
	  **邊**：如果病人的偏好是醫生的約會時段，則在圖中連接這兩個節點。
	- **使用匈牙利算法**：
	  匈牙利算法是一種常見的解決二分圖最大匹配的算法，可以有效地檢查是否存在一個匹配，使得每個病人都能獲得一個約會時段。
- ## Code
  
  ```go
  package main
  
  import "fmt"
  
  // 判斷是否存在一個匹配，使得每個病人都能獲得一個約會時段
  func Solution(A []int, B []int, S int) bool {
      N := len(A)
      
      // 處理邊界情況
      if N == 0 {
          return true
      }
      if S == 0 || N > S {
          return false
      }
      
      // 初始化圖
      graph := make([][]bool, N)
      for i := range graph {
          graph[i] = make([]bool, S)
      }
      
      for i := 0; i < N; i++ {
          if A[i] > S || B[i] > S {
              return false // 如果病人的偏好超出有效的約會時段範圍
          }
          graph[i][A[i]-1] = true
          graph[i][B[i]-1] = true
      }
      
      // 匈牙利算法的匹配結果
      match := make([]int, S)
      for i := range match {
          match[i] = -1
      }
      
      var dfs func(int, []bool) bool
      dfs = func(u int, visited []bool) bool {
          for v := 0; v < S; v++ {
              if graph[u][v] && !visited[v] {
                  visited[v] = true
                  if match[v] == -1 || dfs(match[v], visited) {
                      match[v] = u
                      return true
                  }
              }
          }
          return false
      }
      
      // 嘗試給每個病人分配一個約會時段
      count := 0
      for i := 0; i < N; i++ {
          visited := make([]bool, S)
          if dfs(i, visited) {
              count++
          }
      }
      
      return count == N
  }
  
  func main() {
      A := []int{1, 1, 3}
      B := []int{2, 2, 1}
      S := 3
      fmt.Println(Solution(A, B, S)) // Output: true
  }
  ```
  **圖的建構**：
  根據每個病人的約會偏好，建立一個二分圖。`graph[i][j]` 表示病人 `i` 可以選擇約會時段 `j`。
  **匈牙利算法**：
  `dfs` 函數嘗試找到一個合適的約會時段給每個病人。如果病人 `i` 能夠找到一個未被分配的約會時段，或者找到一個可以重新分配的約會時段，則返回 `true`。
  **匹配計數**：
  嘗試給每個病人分配約會時段，並計數成功分配的病人數量。若分配數量等於病人的數量，則返回 `true`，表示所有病人都能被安排。
- ## Complexity:
  這個解決方案的時間複雜度是 O(N * S^2)，其中 N 是病人的數量，S 是醫生約會時段的數量。對於大規模問題，可能需要進一步優化算法。