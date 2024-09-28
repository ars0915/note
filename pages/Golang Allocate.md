public:: true
tags:: golang, allocate

- ## Introduction
	- 在 Go runtime 進行的記憶體管理會影響效能，編譯器透過 `escape analysis` 決定要把變數放在 stack 或 heap，以下狀況會把變數放到 heap
		- function return pointer
		- 變數的大小在編譯時未知
	- 可以使用 `-benchmem`看 `alloc/op` 知道目前程式的記憶體分配狀況
	- GC 的時候會 STW
- ## Stack and Heap
	- ### Stack
		- 在許多流行的程式設計環境中，`stack` 通常指向 thread 的 `call stack`。
		  `call stack` 是一種 LIFO 堆疊資料結構，用於儲存參數、局部變數以及執行緒執行函數時追蹤的其他資料。每個函數呼叫都會向堆疊新增（推送）一個新幀，每個返回函數都會從堆疊中刪除（彈出）。
		- 當最近的堆疊幀彈出時，我們必須能夠安全地釋放它的記憶體。因此，我們不能在堆疊上儲存以後需要在其他地方引用的任何內容。