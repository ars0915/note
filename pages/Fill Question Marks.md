public:: true
tags:: Algorithm, string, recursive

- ## 題目
	- 一個字串 裡面只會有 a, b, ?，不能有連續3個重複的字元
	  當遇到?時要填入 a 或 b
	  i.e. 
	  "ba?bb" return "baabb"
	  "??abb" return "baabb" or "ababb" or "bbabb"
	  "a?b?aa" return "aabbaa"
- ## Code
  
  ```go
  package main
  
  import (
  	"fmt"
  )
  
  func replaceQuestionMarks(s string) string {
  	result := []byte(s)
  
  	var backtrack func(index int) bool
  	backtrack = func(index int) bool {
  		if index == len(s) {
  			return true
  		}
  
  		if s[index] != '?' {
  			if isValid(string(result[:index+1])) {
  				return backtrack(index + 1)
  			}
  			return false
  		} else {
  			for _, ch := range []byte{'a', 'b'} {
  				result[index] = ch
  				if isValid(string(result[:index+1])) && backtrack(index+1) {
  					return true
  				}
  			}
  			return false
  		}
  	}
  
  	backtrack(0)
  	return string(result)
  }
  
  func isValid(s string) bool {
  	if len(s) >= 3 {
  		last3 := s[len(s)-3:]
  		return !(last3 == "aaa" || last3 == "bbb")
  	}
  	return true
  }
  
  func main() {
  	testCases := []string{"a", "a?bb", "ba?bb", "??abb", "a?b?aa"}
  
  	for _, tc := range testCases {
  		fmt.Printf("Input: %s\n", tc)
  		result := replaceQuestionMarks(tc)
  		fmt.Printf("Output: %s\n\n", result)
  	}
  }
  
  ```
  **backtrack 函數：**用於遞迴地處理每個 ? 字符。對於每個 ?，嘗試用 'a' 或 'b' 替換，並檢查替換後的字串是否有效。
  **isValid 函數：**檢查當前字串是否滿足條件，即不會有三個連續相同的字符。
- ## Complexity:
  最壞情況下，時間複雜度是 **O(2^m)**，其中 **m** 是 `?` 的數量。這是由於每個 `?` 都有兩個選擇，回溯的可能路徑是指數級的。