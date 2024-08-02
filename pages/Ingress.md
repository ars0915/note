public:: true
tags:: Kubernetes

- 用來將服務暴露到cluster外面，並且可以自定義服務的訪問策略
- 當多個 Service 同時運行時，Node 都需要有相對應的 port number 去對應相每個 Service
  ![image.png](../assets/image_1722609839251_0.png)
- ## 與 [[LoadBalancer]] 的差別
	- Ingress 可以解析 L7 的 HTTPS，也能在 ingress 設定 SSL 及 route。
	- 需要 Ingress Controller 來做負載平衡和反向代理。