public:: true
tags:: Kafka, Kafka Partition

- ## Range
	- 以 member_id 的順序分配 `Partition`
	  ![image.png](../assets/image_1727014729534_0.png)
- ## RoundRobin
	- 比 range 更能完全分配，最大化的使用 `Consumer`
	  ![image.png](../assets/image_1727014799185_0.png)
	  當
- ## StickyAssignor
-