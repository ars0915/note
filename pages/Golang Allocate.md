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
		- 在許多流行的程式設計環境中，stack 通常指向 thread 的 call stack。
		  call stack 是一種 LIFO stack 資料結構，用於儲存參數、局部變數以及 thread 執行 function 時追蹤的其他資料。每個 function 呼叫都會向 stack push 一個 frame，每個 return 都會從 stack 中 pop。
		  當最近的 stack frame pop 時，我們必須能夠安全地釋放它的記憶體。因此，我們不能在 stack 上儲存以後需要在其他地方引用的任何內容。
		  由於 thread 由 OS 管理，因此 thread stack 可用的記憶體量通常是固定的，例如在許多 Linux 環境中預設值為 8MB。這意味著我們還需要注意 stack 上最終有多少數據，特別是在 deeply-nested recursive functions 的情況下。如果上圖中的堆疊指標透過堆疊保護，程式將因堆疊溢位錯誤而崩潰。