tags:: Kubernetes

- ## 用途
	- 提供 FQDN 讓其他應用程式存取對應的 Pod，自己會更新容器的 IP
- ## 種類
	- [[ClusterIP]]
	- [[NodePort]]
	- [[LoadBalancer]]
	- [[External]]
- ## Kube-Proxy Mode
	- kube-proxy: 透過`kube-proxy`本身程式內部的邏輯來實現 `service`，由於 `kube-proxy` 是 `user-space` 的應用程式，所以效率非常低落，但是因為是程式化的結果，彈性比較高。
	- iptables: 讓 `kube-proxy` 去維護跟 `service`有關的邏輯部分，真正所有封包轉送都交由 `kernel-space` 的 `iptables` 來處理。 效率比(1)來得強，但是在使用上則是會受限於 `iptables` 的規則與框架。
	- ipvs