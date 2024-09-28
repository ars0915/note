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
		- ![image.png](../assets/image_1727526133129_0.png) 
		  在許多流行的程式設計環境中，stack 通常指向 thread 的 call stack。
		  call stack 是一種 LIFO stack 資料結構，用於儲存參數、局部變數以及 thread 執行 function 時追蹤的其他資料。每個 function 呼叫都會向 stack push 一個 frame，每個 return 都會從 stack 中 pop。
		  當最近的 stack frame pop 時，我們必須能夠安全地釋放它的記憶體。因此，我們不能在 stack 上儲存以後需要在其他地方引用的任何內容。
		  由於 thread 由 OS 管理，因此 thread stack 可用的記憶體量通常是固定的，例如在許多 Linux 環境中預設值為 8MB。這意味著我們還需要注意 stack 上最終有多少數據，特別是在 deeply-nested recursive functions 的情況下。
		  如果上圖中的 stack pointer 超過 stack guard，程式將因 stack overflow 錯誤而 crash。
	- ### Heap
		- 我們可以使用 heap 來儲存程式中所需的資料。此處分配的記憶體不能在 function return 時簡單地釋放，需要仔細管理以避免洩漏和碎片。堆通常會比任何執行緒堆疊大許多倍，大部分最佳化工作將花在研究堆的使用上。