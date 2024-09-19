-
- public:: true
  tags:: Algorithm, string
- ## 題目
	- 求最少刪除幾個字元使字串不是回文，並且只能從兩側刪除
- ## Approach
	- **檢查字串是否是回文**，如果不是回文，則返回 `0`，因為不需要刪除任何字元。
	- **逐步檢查從左右兩側刪除字元的情況**，刪除一個字元、兩個字元等，直到字串變為非回文。
	  在最壞情況下，如果刪除到字串的全部字元還是回文（如 "xxxx"），那麼需要刪除全部的字元。
- ## Code
  
  ```go
  package main
  
  import (
  	"fmt"
  )
  
  // 檢查字串是否為回文
  func isPalindrome(s string) bool {
  	n := len(s)
  	for i := 0; i < n/2; i++ {
  		if s[i] != s[n-1-i] {
  			return false
  		}
  	}
  	return true
  }
  
  // 求最少刪除幾個字元使字串不是回文，並且只能從兩側刪除
  func minDeletionsToNotPalindrome(s string) int {
  	// 空字串直接返回 0
  	if s == "" {
  		return 0
  	}
  
  	// 如果字串本來就不是回文，直接返回 0
  	if !isPalindrome(s) {
  		return 0
  	}
  
  	// 開始嘗試刪除字元，從一個字元開始
  	n := len(s)
  	for i := 1; i <= n; i++ {
  		// 從左側刪除 i 個字元
  		if !isPalindrome(s[i:]) {
  			return i
  		}
  		// 從右側刪除 i 個字元
  		if !isPalindrome(s[:n-i]) {
  			return i
  		}
  	}
  
  	// 如果刪除所有字符後還是回文，那麼返回字串長度
  	return n
  }
  
  func main() {
  	// 測試用例
  	fmt.Println(minDeletionsToNotPalindrome("xyzx"))  // Output: 0
  	fmt.Println(minDeletionsToNotPalindrome("katak")) // Output: 1
  	fmt.Println(minDeletionsToNotPalindrome("xxxx"))  // Output: 4
  	fmt.Println(minDeletionsToNotPalindrome("abcba")) // Output: 1
  }
  ```
  **檢查是否為回文**：如果字串本來不是回文，直接返回 `0`。
  **逐步刪除字元**：從左側或右側開始刪除 1 個字元，然後檢查剩餘部分是否還是回文。如果刪除一側後變成非回文，返回刪除的字元數。繼續這個過程直到找到最少的刪除次數。
  **最壞情況**：如果刪除到所有字元還是回文，那麼返回整個字串的長度，因為最終需要刪除全部字元。