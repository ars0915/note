public:: true
tags:: Kubernetes, Kubernetes Node

- ## 需求
	- 有時會需要介入 pod scheduling，像是
	  1. 讓 pod 被分配到特定的 nodes (例如有特定硬體、node綁IP)
	  2. 希望某些 pod 可以被固定放在一起
	  3. 分散 pod 所屬的 node
	  這些都建立在 **label select** 的基礎上完成
- ## nodeSelector
  id:: 66b38f32-d3a4-4919-86eb-9f22044b23f5
	- ### 為 worker node 指定 label
		- ```shell
		  kubectl label nodes/<node-name> <label-key>=<label-value>
		  ```
	- ### 為 pod spec 設定 nodeSelector
		- ```yaml
		  apiVersion: v1
		  kind: Pod
		  metadata:
		    name: nginx
		    labels:
		      env: test
		  spec:
		    containers:
		    - name: nginx
		      image: nginx
		      imagePullPolicy: IfNotPresent
		    # 設定的地方在 pod.nodeSelector
		    nodeSelector:
		      disk_type: ssd
		  ```
- ## Affinity & Anti-Affinity
	- [nodeSelector](((66b38f32-d3a4-4919-86eb-9f22044b23f5))) 有時無法滿足複雜的需求
	  **affinity/anti-affinity** 加強了幾個地方：
	  1. 可用更彈性的方法來指定多個 label 時的組合，而不再只能用 **AND**
	  2. 以往只能設定是否完全符合條件，現在可以用 `preference(希望可以有，但沒有也沒關係)` 的方式來設定
	  3. 可以指定跟帶有某些 label 的 `pod` 放在一起(or 不要放在一起)，而不是只能指定 worker node label：這樣有助於讓某些 pod 可以被放在同一個 worker node(或不被放在一起)
	- ### Node affinity/anti-affinity
		- node affinity 有以下兩種類型
		  1. **requiredDuringSchedulingIgnoredDuringExecution**
		  2. **preferredDuringSchedulingIgnoredDuringExecution**
			- 上面設定可以拆成三個部份`
			  1. `requiredDuringScheduling`：一定要 node 符合條件，scheduler 才會把 pod 分派到上面去跑，一旦發現不再滿足調度條件，會被驅除
			  2. `preferredDuringScheduling`：會儘量嘗試找尋合適條件的 node，但不強制
			  3. `IgnoredDuringExecution`：表示當 pod 已經正在運作中了，即使 node 的 label 在之後遭到變更，也不會影響正在運作中的 pod
		- 範例：
		  ```yaml
		  apiVersion: v1
		  kind: Pod
		  metadata:
		    name: with-node-affinity
		  spec:
		    affinity:
		      nodeAffinity:
		        # 一定要滿足以下條件才可作 pod scheduling
		        requiredDuringSchedulingIgnoredDuringExecution:
		          # nodeSelector 的條件定義
		          nodeSelectorTerms:
		          - matchExpressions:
		            # node 一定有帶有 以下任何一種 label 才可以
		            # "kubernetes.io/e2e-az-name=e2e-az1"
		            # "kubernetes.io/e2e-az-name=e2e-az2"
		            - key: kubernetes.io/e2e-az-name
		              operator: In
		              values:
		              - e2e-az1
		              - e2e-az2
		        # 儘量滿足以下條件即可作 pod scheduling
		        preferredDuringSchedulingIgnoredDuringExecution:
		        # 這是屬於 prefer 的權重設定(1-100)，符合條件就會得到此權重值
		        # pod 會被分配到最後加總數值最高的 node
		        - weight: 1
		          preference:
		            matchExpressions:
		            # 儘量尋找帶有 label "another-node-label-key=another-node-label-value" 的 node
		            - key: another-node-label-key
		              operator: In
		              values:
		              - another-node-label-value
		    containers:
		    - name: with-node-affinity
		      image: k8s.gcr.io/pause:2.0
		  ```
		- 若同時設定 `nodeSelector` & `nodeAffinity`，那 scheduler 就會尋找同時滿足兩個條件要求的 node
		- 如果在 `nodeAffinity` 中設定多個 `nodeSelectorTerms`，那就只要滿足任何一個 **nodeSelectorTerms** 的要求即可
		- node affinity 目前只有在 pod scheduling 的時候會有用途，當 pod 已經在 node 上運行後，即使 node label 變更了也不會影響正在上面運行中的 pod，除非之後 `requiredDuringSchedulingRequiredDuringExecution` 的功能有推出
		- 在 prefer 的設定中，有個 `weight` 的權重值(1-100)可以設定，而 scheduler 在決定前，還會加上其他 node priority function 來進行綜合考量，最後 pod 會被分配到數值計算結果最高的 node 上去
	- ### inter-pod affinity/anti-affinity
		- 為什麼需要 pod affinity/anti-affinity?
		  跟 ReplicaSets, StatefulSets 或是 Deployments 一起搭配的時候；例如：希望把 workload 分派到特定的 topology 的運行(例如：同一個 node)
		- 有以下兩種設定類型可以使用：
		  1. **requiredDuringSchedulingIgnoredDuringExecution**
		  2. **preferredDuringSchedulingIgnoredDuringExecution**
		-
- ## Taints & Tolerations