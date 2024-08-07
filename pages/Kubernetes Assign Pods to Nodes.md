public:: true
tags:: Kubernetes, Kubernetes Node

- ## 需求
	- 有時會需要介入 pod scheduling，像是
	  1. 讓 pod 被分配到特定的 nodes (例如有特定硬體、node綁IP)
	  2. 希望某些 pod 可以被固定放在一起
	  3. 分散 pod 所屬的 node
	  這些都建立在 **label select** 的基礎上完成
- ## nodeSelector
	- ### 為 worker node 指定 label
		- ```shell
		  kubectl label nodes/<node-name> <label-key>=<label-value>
		  ```
	-
- ## Affinity & Anti-Affinity
- ## Taints & Tolerations