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
			  1. requiredDuringScheduling`：一定要 node 符合條件，scheduler 才會把 pod 分派到上面去跑
			  2. `preferredDuringScheduling`：會儘量嘗試找尋合適條件的 node，但不強制
			  3. `IgnoredDuringExecution`：表示當 pod 已經正在運作中了，即使 node 的 label 在之後遭到變更，也不會影響正在運作中的 pod
		-
	- ### inter-pod affinity/anti-affinity
- ## Taints & Tolerations