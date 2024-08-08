public:: true
tags:: Kubernetes, Kubernetes Node, Kubernetes Pod

- ## What is DaemonSet?
	- DaemonSet 是確保在 Kubernetes 中的每一個 node 上都會有一個指定的 pod 來運行特定的工作，當有新的 node 加入到 Kubernetes cluster 後，系統會自動在那個 node 上長出相同的 DaemonSet pod，當有 node 從 Kubernetes cluster 移除後，該 node 上的 DaemonSet pod 就會自動被清除掉。
	- ### When to use?
		- 日誌收集和監控：當你需要在每個節點上運行一個日誌或監控代理，DaemonSet可以確保每個節點都有這個代理運行，從而集中收集系統日誌或應用程序指標。
		  例如Prometheus監控集群，在每個節點上都運行一個node-exporter來收集監控節點的訊息，或是fluented, logstash。在每個節點上運行以收集容器的日誌。
		-
- ## How Daemon Pods are scheduled?
- ## Taints and Tolerations with DaemonSet
- ## Communicating with Daemon Pods