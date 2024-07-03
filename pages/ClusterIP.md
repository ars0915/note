tags:: Kubernetes, Kubernetes Service

- ## 目標
	- 透過 FQDN 存取到目標容器
	- 只有 `Cluster` 內的應用程式/節點才可以存取
	- 有多個 `Endpoints`如何處理選擇
- ## Access By FQDN
	- 只有`kube-dns`能夠理解`Service`對應的FQDN，``
-