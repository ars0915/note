public:: true
tags:: Kubernetes, Kubernetes Node, Kubernetes Pod

- ## What is DaemonSet?
	- DaemonSet 是確保在 Kubernetes 中的每一個 node 上都會有一個指定的 pod 來運行特定的工作，當有新的 node 加入到 Kubernetes cluster 後，系統會自動在那個 node 上長出相同的 DaemonSet pod，當有 node 從 Kubernetes cluster 移除後，該 node 上的 DaemonSet pod 就會自動被清除掉。
	- ### When to use?
		- 日誌收集和監控：當你需要在每個節點上運行一個日誌或監控代理，例如 Prometheus 監控集群，在每個節點上都運行一個node-exporter來收集監控節點的訊息，或是 fluented, logstash。在每個節點上運行以收集容器的日誌。
		- 網路代理：對於需要節點級網路代理的應用程序，如CNI，你可以使用DaemonSet確保每個節點都運行這個代理。例如 flannel, calico。
		- 分佈式存儲：當你需要在每個節點上運行分佈式存儲代理，以實現數據持久性和可用性。例如 glusterd, ceph 要部署在每個節點上以提供持久性儲存。
		- 應用程式資料庫：對於某些應用程式，每個節點可能需要本地存儲或快取資料庫的副本。
		- Node-Level操作：當你需要執行僅涉及單個節點的操作時，如特定節點的升級或清理。
		-
- ## How Daemon Pods are scheduled?
- ## Taints and Tolerations with DaemonSet
- ## Communicating with Daemon Pods