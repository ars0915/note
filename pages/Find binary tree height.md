public:: true
tags:: Algorithm, Binary Tree

-
- ## Approach
	- 透過遞迴找到左右子樹的高度最大值，然後再加上目前的節點 (+1)
- ## Code
	- ```go
	  func Solution(T *Tree) int {
	      if T == nil {
	          return -1 // The height of an empty tree is typically -1 (or 0 in some cases)
	      }
	      
	      // Recursively find the height of the left and right subtrees
	      leftHeight := Solution(T.L)
	      rightHeight := Solution(T.R)
	      
	      // The height of the tree is the maximum of the two subtrees' heights, plus 1
	      return max(leftHeight, rightHeight) + 1
	  }
	  
	  // Helper function to get the maximum of two integers
	  func max(a, b int) int {
	      if a > b {
	          return a
	      }
	      return b
	  }
	  
	  ```
- ## Example
	- ```markdown
	        1
	       / \
	      2   3
	     / \
	    4   5
	  
	  ```
	  高度為2，因為從根到葉最長的距離有兩個邊
- ## Time Complexity
  **O(n)**: The algorithm visits each node once.
- ## Space Complexity
  **O(h)**: The recursion depth is equal to the height of the tree, so in the worst case, the space complexity is proportional to the height `h` of the tree, which is O(log n) for a balanced tree and O(n) for a skewed tree.