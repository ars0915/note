public:: true
tags:: Kubernetes, Kubernetes Node, Kubernetes Pod

- # What is StatefulSet?
	- 基本上 StatefulSet 中在 pod 的管理上都是與 Deployment 相同，基於相同的 container spec 來進行；而其中的差別在於 **StatefulSet controller 會為每一個 pod 產生一個固定的識別資訊，不會因為 pod reschedule 後有變動**。
	- ## When to use?
		- 需要穩定的 persistent storage (pod reschedule 後還是能存取到相同的資料，基本上用 PVC 就可以解決)
		- 需要穩定 & 唯一的網路識別 (pod reschedule 後的 pod name & hostname 都不會變動)
		- 佈署 & scale out 的時後，每個 pod 的產生都是有其順序且逐一慢慢完成的
	- ## 限制
		- storage 的部份一定要綁定 PVC，並綁定到特定的 StorageClass or 預先配置好的 PersistentVolume，確保 pod 被刪除後資料依然存在
		- 需要額外定義一個 [[Headless Service]] 與 StatefulSet 搭配，確保 pod 有固定的 network identity
	- ## Example
	  ![image.png](../assets/image_1723717561080_0.png)
	  
	  ```yaml
	  ---
	  # v1.9 版本之前必須使用 "apps/v1beta2"
	  apiVersion: apps/v1
	  kind: StatefulSet
	  metadata:
	    name: web
	  spec:
	    selector:
	      # 必須與 ".spec.template.metadata.labels" 相同
	      matchLabels:
	        app: nginx
	    serviceName: "nginx"
	    replicas: 3
	    template:
	      metadata:
	        # 必須與 ".spec.selector.matchLabels" 相同
	        labels:
	          app: nginx
	      spec:
	        # 每個pod依順序性刪除時間隔時間
	        terminationGracePeriodSeconds: 10
	        containers:
	        - name: nginx
	          image: k8s.gcr.io/nginx-slim:0.8
	          ports:
	          - containerPort: 80
	            name: web
	          # 指定將 pvc 掛載到特定的目錄上
	          volumeMounts:
	          - name: www
	            mountPath: /usr/share/nginx/html
	    # 使用 persistent volume 來確保資料不會因為 pod reschedule 而消失
	    # 以下是使用 volumeClaimTemplates + StorageClass 來完成
	    volumeClaimTemplates:
	    - metadata:
	        name: www
	      spec:
	        accessModes: [ "ReadWriteOnce" ]
	        # 透過動態方式自動產生PVC，這裡需要指定一個已經存在並且合法的storageClass
	        storageClassName: "my-gfs-storageclass"
	        resources:
	          requests:
	            storage: 1Gi
	  ```
- # 如何識別 StatefulSet 產生的 Pod?
	- ## Ordinal Index
		- 若一個 statefulset 包含了 N 個 replica，那每一個 pod 都會被分配到一個獨一無二的索引，從 `0` ~ `N-1`，即使 pod reschedule 也不會改變。
	- ## Stable Network ID
		- 每個在 statefulset 中的 pod 都會有自己獨一無二的 hostname，命名的規則為 `$(statefulset name)-$(ordinal index)`
		  statefulset 還會透過 Headless Service 來維持 pod domain name 是固定指到 pod IP，並使用以下的標準格式存取 domain name：
		  
		  `$(service name).$(namespace).svc.cluster.local`
		  
		  其中 `cluster.local` 是當初安裝 k8s 所設定的 cluster domain，若安裝時有修改的話，上面的 domain name 也必須跟著調整。
	- ## Stable Storage
		- 若是 statefulset 中的 replicas 設定為大於 1，為了確保每個 pod 在產生時都會有各自對應的 persistent storage 可用，在 Storage 的部份就要以 `volumeClaimTemplates` + `StorageClass` 來設定
		  
		  
		  ```yaml
		  # 宣告 volumeClaimTemplates
		  volumeClaimTemplates:
		  - metadata:
		      name: www
		    spec:
		      accessModes: [ "ReadWriteOnce" ]
		      # 指定搭配的 StorageClass
		      storageClassName: "my-gfs-storageclass"
		      resources:
		        requests:
		          # 容量需求為 1GB
		          storage: 1Gi
		  ```
		  
		  透過以上的設定，statefulset 中每個 pod 副本產生時，k8s 會自動執行以下工作：
			- 1. StorageClass “my-gfs-storageclass“ 會負責對特定的 storage 要求 1GB 的空間
			- 2. StorageClass “my-gfs-storageclass“ 動態產生 persistent volume(PV)，並與上面的空間綁定
			- 3. 透過 volumeClaimTemplates 為每個 pod 產生一個 persistent volume claim(PVC)，並與步驟 2 的 PV 綁定
			- 4. pod 使用 PVC 並掛載到指定的目錄上
		- 透過 StorageClass，可以根據 resource 的需求，產生 persistent volume 並與特定的 storage 綁定
		  PV 不會因為 pod 被刪除 or reschedule 而消失(只能手動刪除)，如此才能達成 statefulset 對 storage 的要求。
	- ## Pod Name Label
		- statefulset 產生每個 pod 時，都會自動幫 pod 加上名稱為 `statefulset.kubernetes.io/pod-name` 的 label，而 label value 就是上面提到的 pod name
- # Deployment & Scaling 流程說明
	- ## 佈署 & Scale out
	- ## 刪除 & Scale in
	- ## Pod Management Policy
- # 更新(update)要如何進行?
	- ## On Delete
	- ## Rolling Update
		- ### Partition