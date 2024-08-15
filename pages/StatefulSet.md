public:: true
tags:: Kubernetes, Kubernetes Node, Kubernetes Pod

- # What is StatefulSet?
	- 基本上 StatefulSet 中在 pod 的管理上都是與 Deployment 相同，基於相同的 container spec 來進行；而其中的差別在於 **StatefulSet controller 會為每一個 pod 產生一個固定的識別資訊，不會因為 pod reschedule 後有變動**。
	-
- ## Headless Service
  https://ithelp.ithome.com.tw/articles/10251596