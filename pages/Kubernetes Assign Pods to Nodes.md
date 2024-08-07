public:: true
tags:: Kubernetes, Kubernetes Node, Kubernetes Pod

- # 需求
	- 有時會需要介入 pod scheduling，像是
	  1. 讓 pod 被分配到特定的 nodes (例如有特定硬體、node綁IP)
	  2. 希望某些 pod 可以被固定放在一起
	  3. 分散 pod 所屬的 node
	  這些都建立在 **label select** 的基礎上完成
- # nodeSelector
  id:: 66b38f32-d3a4-4919-86eb-9f22044b23f5
	- ## 為 worker node 指定 label
		- ```shell
		  kubectl label nodes/<node-name> <label-key>=<label-value>
		  ```
	- ## 為 pod spec 設定 nodeSelector
	  ```yaml
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
	- ## Built-in Node Labels
	  id:: 66b3944a-73aa-4ab5-a17e-6bd3caa5df69
		- 每個 node 都會有一些內建的 label set，像是：
		  beta.kubernetes.io/arch
		  beta.kubernetes.io/os
		  kubernetes.io/hostname
		  node-role.kubernetes.io/master
		  node-role.kubernetes.io/node
		  這些內建的 node label 被稱為 `topologyKey`
- # Affinity & Anti-Affinity
	- [nodeSelector](((66b38f32-d3a4-4919-86eb-9f22044b23f5))) 有時無法滿足複雜的需求
	  **affinity/anti-affinity** 加強了幾個地方：
	  1. 可用更彈性的方法來指定多個 label 時的組合，而不再只能用 **AND**
	  2. 以往只能設定是否完全符合條件，現在可以用 `preference(希望可以有，但沒有也沒關係)` 的方式來設定
	  3. 可以指定跟帶有某些 label 的 `pod` 放在一起(or 不要放在一起)，而不是只能指定 worker node label：這樣有助於讓某些 pod 可以被放在同一個 worker node(或不被放在一起)
	- ## Node affinity/anti-affinity
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
	- ## inter-pod affinity/anti-affinity
		- 為什麼需要 pod affinity/anti-affinity?
		  跟 ReplicaSets, StatefulSets 或是 Deployments 一起搭配的時候；例如：希望把 workload 分派到特定的 topology 的運行(例如：同一個 node)
		- 有以下兩種設定類型可以使用：
		  1. **requiredDuringSchedulingIgnoredDuringExecution**
		  2. **preferredDuringSchedulingIgnoredDuringExecution**
		- 當實際要設定 pod affinity/anti-affinity，有兩個條件可以用來進行設定：
		  **pod label set**：這個部份其實很清楚，跟 node label set 其實是一樣的東西
		  **node topology key**((66b3944a-73aa-4ab5-a17e-6bd3caa5df69))：這個部份通常就是 k8s 的 build-in node label (表示希望檢查 worker node 是否也符合條件)
		  topologyKey 的設定用意在於，如果有多個 pod 需要分配，並指定了 topologyKey，那 scheduler 在分配時就**不可以**多個 pod 放到帶有相同 value 的topologyKey(Label) 的 node 上
		- #### 範例：
		  假設 k8s cluster 中有 3 個 worker node(node03, node04, node05)，且要在裡面安裝一個三份 replica 的 web application，而這個 web application 使用到 redis 作為 cache，為了平均的分散 work load，希望可以將 pod 佈署成：
		  node03: redis + web-app
		  node04: redis + web-app
		  node05: redis + web-app
			- 佈署 redis，並確保分散在不同的 node
			  ```yaml
			  apiVersion: apps/v1
			  kind: Deployment
			  metadata:
			    name: redis-cache
			  spec:
			    selector:
			      matchLabels:
			        app: store
			    replicas: 3
			    template:
			      metadata:
			        labels:
			          app: store
			      spec:
			        affinity:
			          # 確保 pod 不會落在帶有 app in "store" label 的 pod 所在的 worker node 上
			          podAntiAffinity:
			            requiredDuringSchedulingIgnoredDuringExecution:
			            - labelSelector:
			                matchExpressions:
			                - key: app
			                  operator: In
			                  values:
			                  - store
			              # pod 在分配時要確保 worker node 帶有名稱為 "kubernetes.io/hostname" 的 label
			              # 但不能有相同的 value
			              topologyKey: "kubernetes.io/hostname"
			        containers:
			        - name: redis-server
			          image: redis:3.2-alpine
			  ```
			- 佈署 web app，並跟 redis 放一起
			  ```yaml
			  apiVersion: apps/v1
			  kind: Deployment
			  metadata:
			    name: web-server
			  spec:
			    selector:
			      matchLabels:
			        app: web-store
			    replicas: 3
			    template:
			      metadata:
			        labels:
			          app: web-store
			      spec:
			        affinity:
			          # 跟佈署 redis 相同，不要讓 web app 分派到同樣的 worker node 上
			          podAntiAffinity:
			            requiredDuringSchedulingIgnoredDuringExecution:
			            - labelSelector:
			                matchExpressions:
			                - key: app
			                  operator: In
			                  values:
			                  - web-store
			              topologyKey: "kubernetes.io/hostname"
			          # 這是跟 redis cache 可以一對一放在一起並分散到不同 worker node 的關鍵
			          podAffinity:
			            requiredDuringSchedulingIgnoredDuringExecution:
			            # 在這裡設定 redis cache 的 label set
			            # 指定要找到 redis 所在的 worker node 來放 web app pod
			            - labelSelector:
			                matchExpressions:
			                - key: app
			                  operator: In
			                  values:
			                  - store
			              # 但是要分散到不同的 worker node 上
			              # (需要帶有 kubernetes.io/hostname label 的 worker node，但 value 不能相同)
			              topologyKey: "kubernetes.io/hostname"
			        containers:
			        - name: web-app
			          image: nginx:1.12-alpine
			  ```
- # Taints & Tolerations
  id:: 66b42b6b-71d0-47af-b499-7c719b63b5ed
	- `taint`：設計讓 pod 如何不要被分派到某個 worker node
	  與 `toleration` 共同搭配使用，目的就是要避免讓 pod 被分派到不正確 or 不合適的 worker node 上，運作原理大概如下：
	  如果有特定的 node 被加上了 taint(汙點)，pod 就不會被分派到上面，除非 pod spec 有設定 toleration(容忍) 來接受這些 taint (必須全部 taint 都接受才行)
	- ## 如何設定 node taint
		- 每個 taint 都有以下 3 個屬性：
		  1. Key
		  2. Value
		  3. Effect：共有三種，分別是 **NoSchedule**, **PreferNoSchedule** & **NoExecute**
			- #### NoSchedule
			  假設最後某個 node 上留下的 taint 的 effect 為 `NoSchedule`，那 k8s 就不會把該 pod 分派到該 node 上，但不影響正在運作中的 pod。
			- #### PreferNoSchedule
			  假設最後某個 node 上留下的 taint 的 effect 為 PreferNoSchedule，那 k8s 就儘量不會把該 pod 分派到該 node 上(最後要是沒辦法的時候還是會破功)，但不影響正在運作中的 pod。
			- #### NoExecute
			  假設某個 node 被設定了 effect 為 NoExecute 的 taint，那 k8s 還會把已經存在該 node 上的 pod 趕走，也不會把該 pod 分派到該 node 上。
			  
			  設定 `tolerationSeconds` 可以表示在 taint 被增加之後，帶有相對應 toleration 的 pod 還可以在該 node 上存在多久
		- 設定 node taint
		  ```
		  kubectl taint nodes node1 key=value:NoSchedule
		  ```
		  移除 node taint
		  在 Key:Effect 後面加上 -
		  ```
		  kubectl taint nodes node1 key:NoSchedule-
		  ```
	- ## 設定 pod toleration
		- ```yaml
		  # 表示可以接受"帶有 key=value & effect=NoSchedule" 的 taint
		  tolerations:
		  - key: "key"
		    operator: "Equal"
		    value: "value"
		    effect: "NoSchedule"
		  ```
		  也可以不設定 value
		  
		  ```yaml
		  # 表示可以接受"存在 key(不論 value 為何) & effect=NoSchedule" 的 taint
		  tolerations:
		  - key: "key"
		    operator: "Exists"
		    effect: "NoSchedule"
		  ```
	- ## Taint based Evictions
		- k8s 中有 node controller 會持續監控每個 node 的狀態並回報，因此當它發現某些 node 有狀況時，可以透過為這個 node 增加 taint 的方式，將上面正在運作的 pod 驅離到其他 node 上去執行
			- `node.kubernetes.io/not-read`: Node尚未準備好，相當於Node status Ready為False。
			  `node.kubernetes.io/unreachable`: Node Controller訪問不到Node，相當於Node status Ready為Unknown。
			  `node.kubernetes.io/out-of-disk`: Node的Storage耗盡。
			  `node.kubernetes.io/memory-pressure`: Node的內存面臨壓力。
			  `node.kubernetes.io/disk-pressure`: Node的Storage面臨壓力。
			  `node.kubernetes.io/network-unavailable`: Node的network無法使用。
			  `node.kubernetes.io/unschedulable`: Node無法被調度。
			  `node.cloudprovider.kubernetes.io/uninitialized`: 若透過kubectl指定了一個"外部" 雲平台驅動， 它將給當前節點添加一個污點將其標誌為不可用。在cloud-controller-manager 的一個控制器初始化這個節點後，kubelet 將刪除這個污點。
			- 除了在該 node 上增加一個 taint 的設定外，還會在該 node 上的每個 pod 加上相對應的 toleration 設定，並設定 tolerationSeconds=300，這表示每個 pod 都還可以留在該 node 上 5 分鐘。
			  tolerationSeconds 的設定是 k8s 自動給進去的，若要更改預設值 300，可以透過 DefaultTolerationSeconds admission controller
		- 此外，要讓 taint based eviction 運作，必須要開啟 `TaintBasedEvictions` feature gate (預設是關閉的)
		- 設定這功能需要注意 node controller 中的 `--node-eviction-rate`(預設 0.1，表示 10 秒驅離一個 pod) 設定，必須很小心的設定這個值，避免大規模驅離 pod 時造成整個 cluster 崩潰的連鎖效應 (特別是當 master node 發生崩潰的時候)
	- ## Taint Nodes by Condition
		- 希望當 node 發生特定問題的時候，不要自動被加上 taint，避免 pod 無法被分派到該 node
		- 在 v1.12 後，`TaintNodesByCondition` feature 已經變成 beta 功能且預設開啟，管理者可以透過這個功能選擇性的忽略某些 node 的狀況，而 `TaintNodesByCondition` 實際運作狀況是這樣的：
		  
		  1. 若 node 發生狀況時，k8s 會自動加上 Effect=NoSchedule 的 taint 到該 node 上
		  2. 若 `TaintNodesByCondition` 有設定忽略特定狀況，就不會加上 taint