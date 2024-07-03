tags:: Kubernetes

- ## 用途
	- 提供 FQDN 讓其他應用程式存取對應的 Pod，自己會更新容器的 IP
- ## 種類
	- [[ClusterIP]]
	- [[NodePort]]
	- [[LoadBalancer]]
	- [[External]]
- ## Kube-Proxy Mode
	- kube-proxy
	- iptables
	- ipvs