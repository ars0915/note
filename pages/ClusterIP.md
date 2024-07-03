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
	  `Cluster-IP`是一個虛擬IP，針對該IP發送的封包在滿足**特定條件**後會透過DNAT轉換到其中一個`Endpoints`
	  ```
	  vortex-dev:05:43:50 [~/go/src/github.com/hwchiu/kubeDemo](master)vagrant
	  $sudo iptables-save -t nat | grep nginx-cluster | grep DNAT
	  -A KUBE-SEP-7MBJVYFMXTKOJUKD -p tcp -m comment --comment "default/k8s-nginx-cluster:" -m tcp -j DNAT --to-destination 10.244.0.88:80
	  -A KUBE-SEP-ARZAHNE3T3EMMTGB -p tcp -m comment --comment "default/k8s-nginx-cluster:" -m tcp -j DNAT --to-destination 10.244.0.90:80
	  -A KUBE-SEP-O3CWA7STMVCKFPRY -p tcp -m comment --comment "default/k8s-nginx-cluster:" -m tcp -j DNAT --to-destination 10.244.0.89:80
	  
	  vortex-dev:05:43:54 [~/go/src/github.com/hwchiu/kubeDemo](master)vagrant
	  $kubectl get endpoints k8s-nginx-cluster
	  NAME                ENDPOINTS                                      AGE
	  k8s-nginx-cluster   10.244.0.88:80,10.244.0.89:80,10.244.0.90:80   1d
	  ```
- ## Cluster Only
	- 上面提到的特定條件就是**只有``Cluster``內的應用程式/節點可以存取**，使用`iptables build-in chain`的 `OUTPUT/PREROUTING`來達成
	  OUTPUT: 本地節點送出的封包會先到這
	  PREROUTING: 本地
	-