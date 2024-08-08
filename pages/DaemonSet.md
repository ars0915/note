public:: true
tags:: Kubernetes, Kubernetes Node, Kubernetes Pod

- ## What is DaemonSet?
	- DaemonSet 是確保在 Kubernetes 中的每一個 node 上都會有一個指定的 pod 來運行特定的工作，當有新的 node 加入到 Kubernetes cluster 後，系統會自動在那個 node 上長出相同的 DaemonSet pod，當有 node 從 Kubernetes cluster 移除後，該 node 上的 DaemonSet pod 就會自動被清除掉。
	- ### When to use?
		- 日誌收集和監控：當你需要在每個節點上運行一個日誌或監控代理，例如 Prometheus 監控集群，在每個節點上都運行一個node-exporter來收集監控節點的訊息，或是 fluented, logstash。在每個節點上運行以收集容器的日誌。
		- 網路代理：對於需要節點級網路代理的應用程序，如CNI，你可以使用DaemonSet確保每個節點都運行這個代理。例如 flannel, calico。
		- 分佈式存儲：當你需要在每個節點上運行分佈式存儲代理，以實現數據持久性和可用性。例如 glusterd, ceph 要部署在每個節點上以提供持久性儲存。
		- 應用程式資料庫：對於某些應用程式，每個節點可能需要本地存儲或快取資料庫的副本。
		- Node-Level操作：當你需要執行僅涉及單個節點的操作時，如特定節點的升級或清理。
	- ### 範例
	  ```yaml
	  apiVersion: apps/v1
	  kind: DaemonSet
	  metadata:
	    name: fluentd-elasticsearch
	    namespace: kube-system
	    labels:
	      k8s-app: fluentd-logging
	  spec:
	    # .spec.selector 一旦定義後就無法再變更了
	    # 必須與 .spec.template.metadata.labels 的定義相同
	    # 這裡可以使用 matchLabels 或是 matchExpressions(用來處理較為複雜的 label 組合)
	    selector:
	      matchLabels:
	        name: fluentd-elasticsearch
	    # 此為必要欄位，與 pod template 相同
	    # 用來定義 DaemonSet 的內容應該要長什麼樣子
	    template:
	      metadata:
	        labels:
	          name: fluentd-elasticsearch
	      # 由於是 DaemonSet 的關係，因此 .spec.template.spec.restartPolicy 永遠是 "Always"
	      # 預設值為 "Always"
	      spec:
	        tolerations:
	        - key: node-role.kubernetes.io/master
	          effect: NoSchedule
	        containers:
	        - name: fluentd-elasticsearch
	          image: k8s.gcr.io/fluentd-elasticsearch:1.20
	          resources:
	            limits:
	              memory: 200Mi
	            requests:
	              cpu: 100m
	              memory: 200Mi
	          volumeMounts:
	          - name: varlog
	            mountPath: /var/log
	          - name: varlibdockercontainers
	            mountPath: /var/lib/docker/containers
	            readOnly: true
	        terminationGracePeriodSeconds: 30
	        volumes:
	        - name: varlog
	          hostPath:
	            path: /var/log
	        - name: varlibdockercontainers
	          hostPath:
	            path: /var/lib/docker/containers
	  ```
	  **注意**
	  1. `.spec.selector` 一旦定義後就不建議再修改，若是修改可能會造成某些 pod 變成孤兒(因為 DaemonSet controller 無法管理到正確的 pod)
	  2. `.spec.selector` 的定義必須與 .spec.template.metadata.labels 相同，否則會被 API server 拒絕套用
	  3. 不可以再建立(或是透過其他 controller 建立，例如： [[Deployment]] )帶有與 DaemonSet 相同 label 組合的 pod，否則會被 DaemonSet 認為是自己所產生的
- ## How Daemon Pods are scheduled?
	-
- ## Taints and Tolerations with DaemonSet
- ## Communicating with Daemon Pods