public:: true
tags:: golang, allocate

- ## Introduction
	- 在 Go runtime 進行的記憶體管理會影響效能，編譯器透過 `escape analysis` 決定要把變數放在 stack 或 heap，以下狀況會把變數放到 heap
		- function return pointer
		- 變數的大小在編譯時未知
	- 可以使用 `-benchmem`看 `alloc/op` 知道目前程式的記憶體分配狀況
	- GC 的時候會 STW