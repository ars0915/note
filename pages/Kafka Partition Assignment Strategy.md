public:: true
tags:: Kafka, Kafka Partition

- apache Kafka的重平衡（rebalance），一直以來都是人詬病。因為重平衡過程會觸發stop-the-world（STW），此時對應topic的資源都會處於不可用的狀態。小規模的集群還好，如果是大規模的集群，例如幾百個節點的consumer或kafka connect等，那麼重平衡就是一場災難
- ## Range
  id:: 66f0267f-0f03-48bf-91e4-5702b44311df
	- 以 member_id 的順序分配 `Partition`
	  ![image.png](../assets/image_1727014729534_0.png)
- ## RoundRobin
  id:: 66f02700-7a4d-495f-a0b9-3d5da538f47c
	- 比 [Range](((66f0267f-0f03-48bf-91e4-5702b44311df))) 更能完全分配，最大化的使用 `Consumer`
	  ![image.png](../assets/image_1727014799185_0.png)
	  當 `Comsumer` 有變動時，需要重新分配的 `Partition` 會影響蠻大的
	  ![image.png](../assets/image_1727014942717_0.png)
- ## StickyAssignor
	- 和 [RoundRobin](((66f02700-7a4d-495f-a0b9-3d5da538f47c))) 的分配方式相似，但減少 rebalance 的變動
	  ![image.png](../assets/image_1727015050936_0.png)