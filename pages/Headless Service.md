public:: true
tags:: Kubernetes, Kubernetes Service

- #
- 將 spec.clusterIP 設定成 None。
  
  因為沒有ClusterIP，kube-proxy 並不處理此類服務，因為沒有load balancing或proxy 代理設置，在訪問服務的時候回返回後端的全部的Pods IP位址，主要用於開發者自己根據pods進行負載平衡器的開發(設定了selector)。
-