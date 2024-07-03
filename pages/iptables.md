tags:: Kubernetes

- 當封包有 Match 規則條件，即執行 Action(Target)
- 每條規則都屬於一個`chain`，一個`chain`可以有多條規則
- PREROUTING
- FORWARD
- POSTRING
-
- ## Enpoints
- 在 Kubernetes 中`endpoints`代表實際上 `POD` 的 `IP`
	- ```
	  vortex-dev:02:00:28 [~/go/src/github.com/hwchiu/kubeDemo](master)vagrant
	  $kubectl get endpoints
	  NAME                ENDPOINTS                                      AGE
	  k8s-nginx-cluster   10.244.0.88:80,10.244.0.89:80,10.244.0.90:80   9h
	  k8s-nginx-node      10.244.0.88:80,10.244.0.89:80,10.244.0.90:80   9h
	  kubernetes          172.17.8.100:6443                              11d
	  ```