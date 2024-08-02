public:: true
tags:: Kubernetes

- 用來將服務暴露到cluster外面，並且可以自定義服務的訪問策略
- ## 與 [[LoadBalancer]] 的差別
	- Ingress 可以解析 L7 的 HTTPS，也能在 ingress 設定 SSL 及 route。
	- 需要 Ingress Controller 來做負載平衡和反向代理。