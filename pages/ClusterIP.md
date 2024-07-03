tags:: Kubernetes, Kubernetes Service

- ## 目標
	- 透過 FQDN 存取到目標容器
	- 只有 `Cluster` 內的應用程式/節點才可以存取
	- 有多個 `Endpoints`如何處理選擇
- ## Access By FQDN
	- 只有`kube-dns`能夠理解`Service`對應的FQDN
	  ```
	  vortex-dev:05:36:40 [~/go/src/github.com/hwchiu/kubeDemo](master)vagrant
	  $kubectl get svc
	  NAME                TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)        AGE
	  k8s-nginx-cluster   ClusterIP   10.98.51.150   <none>        80/TCP         1d
	  k8s-nginx-node      NodePort    10.99.157.45   <none>        80:32293/TCP   1d
	  ```
-