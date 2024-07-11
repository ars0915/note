public:: true
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
	- [[iptables]]: 讓 `kube-proxy` 去維護跟 `service`有關的邏輯部分，真正所有封包轉送都交由 `kernel-space` 的 `iptables` 來處理。 效率比`kube-proxy`來得強，但是在使用上則是會受限於 `iptables` 的規則與框架。
	- ipvs: (IP virtual switch) 與`iptables`類似，只是在 `kernel-space` 裡面採用 `ipvs` 的方式來轉送封包，相對於 `iptables` 本身效率更高，同時也不會受限於`iptables` 的使用規則
	- 可以藉由設定 `kube-proxy` 裡面的 `--proxy-mode` 這個參數來決定要使用哪一種實現方式，kube-proxy 本身會透過 `daemonset` 的方式部屬到每一個節點上
-