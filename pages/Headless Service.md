public:: true
tags:: Kubernetes, Kubernetes Service

- # What is Headless Service
	- **將 spec.clusterIP 設定成 None**。
	- 沒有 ClusterIP，因此存取 service 時，k8s DNS 就沒有任何 ClusterIP 的資訊可以回應給 client
	- 若有搭配 Label Selectors，k8s 就會建立相對應的 endpoint，而存取 service 時，k8s DNS 就會直接回應 endpoint list 的資訊(A record)，因此 client 可以使用 service domain name 直接存取到 pod，這也意外著headless service並沒有負載平衡。
	- 因為沒有ClusterIP，kube-proxy 並不處理此類服務，因為沒有 load balancing 或 proxy 代理設置，在訪問服務的時候回返回後端的全部的Pods IP位址，主要用於開發者自己根據pods進行負載平衡器的開發(設定了selector)。
-