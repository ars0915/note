public:: true
tags:: Kubernetes, Kubernetes Service, iptables

- `NodePort` 本身包含了 [[ClusterIP]] 的能力，也可以讓`Cluster`外部透過指定的Port存取`Cluster`內的應用程式。
- ## 實現方式
  根據 [[ClusterIP]]的流程中得知，要支援`NodePort`只需要修改前面的條件
  1. 封包若來自叢集上的應用程式/節點，則跳到 `KUBE-SVC`
  2. 如果封包的目標IP地址是 ClusterIP 所提供的虛擬IP地址, 則跳到 KUBE-SVC-XXXX
-