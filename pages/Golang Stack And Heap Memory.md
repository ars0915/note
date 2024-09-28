public:: true
tags:: golang, allocate

- ## Introduction
	- 在 Go runtime 進行的記憶體管理會影響效能，傳出 pointer 或 struct 讓編譯器決定要把變數放在 stack 或 heap
	- 可以使用 `-benchmem`看 `alloc/op` 知道目前程式的記憶體分配狀況
	- GC 的