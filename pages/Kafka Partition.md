public:: true
tags:: Kafka

- `Partition`（分區）是 [[Kafka]] 的核心角色，對於 [[Kafka]] 的存儲結構、消息的生產消費方式都至關重要。
- ## Events, Streams, Topics
	- 在深入 `Partition` 之前，我們先看幾個更高層次的概念，以及它們與 `Partition` 的聯繫。
	  `Event`（事件）代表過去發生的一個事實。簡單理解就是一條消息、一條記錄。
	  `Event` 是不可變的，但是很活躍，經常從一個地方流向另一個地方。
	  `Stream` 事件流表示運動中的相關事件。
	  當一個事件流進入 [[Kafka]] 之後，它就成爲了一個 `Topic` 主題。
	  ![image.png](../assets/image_1726931716992_0.png)
	  所以，`Topic` 就是具體的事件流，也可以理解爲一個 `Topic` 就是一個靜止的 `Stream`。
	  `Topic` 把相關的 `Event` 組織在一起，並且保存。一個 `Topic` 就像數據庫中的一張表。
- ## Partition 分區
	- ![image.png](../assets/image_1726931835692_0.png)
	  [[Kafka]] 中 `Topic` 被分成多個 `Partition` 分區。
	  `Topic` 是一個邏輯概念，`Partition` 是最小的存儲單元，掌握着一個 `Topic` 的部分數據。
	  每個 `Partition` 都是一個單獨的 log 文件，每條記錄都以追加的形式寫入。
	  ![image.png](../assets/image_1726931865401_0.png)
	  `Record`（記錄） 和 `Message`（消息）是一個概念。
- ## Offsets（偏移量）和消息的順序
	- `Partition` 中的每條記錄都會被分配一個唯一的序號，稱爲 `Offset`（偏移量）。
	  `Offset` 是一個遞增的、不可變的數字，由 [[Kafka]] 自動維護。
	  當一條記錄寫入 `Partition` 的時候，它就被追加到 log 文件的末尾，並被分配一個序號，作爲 `Offset`。
	  ![image.png](../assets/image_1726931922172_0.png)
	  如上圖，這個 `Topic` 有 3 個 `Partition` 分區，向 `Topic` 發送消息的時候，實際上是被寫入某一個 `Partition`，並賦予 `Offset`。
	  消息的順序性需要注意，一個 `Topic` 如果有多個 `Partition` 的話，那麼從 `Topic` 這個層面來看，消息是無序的。
	  但單獨看 `Partition` 的話，`Partition` 內部消息是有序的。
	  所以，一個 `Partition` 內部消息有序，一個 `Topic` 跨 `Partition` 是無序的。
	  如果強制要求 `Topic` 整體有序，就只能讓 `Topic` 只有一個 `Partition`或是另外在 Application 處理。
- ## Partition 爲 Kafka 提供了擴展能力
	- ![image.png](../assets/image_1726932000561_0.png)
	  一個 [[Kafka]] 集羣由多個 `Broker`（就是 Server） 構成，每個 `Broker` 中含有集羣的部分數據。
	  [[Kafka]] 把 `Topic` 的多個 `Partition` 分佈在多個 `Broker` 中。
	  
	  這樣會有多種好處：
		- 如果把 `Topic` 的所有 `Partition` 都放在一個 `Broker` 上，那麼這個 `Topic` 的可擴展性就大大降低了，會受限於這個 `Broker` 的 IO 能力。把 `Partition` 分散開之後，`Topic` 就可以水平擴展 。
		- 一個 `Topic` 可以被多個 `Consumer` 並行消費。如果 `Topic` 的所有 `Partition` 都在一個 `Broker`，那麼支持的 `Consumer` 數量就有限，而分散之後，可以支持更多的 `Consumer`。
		- 一個 `Consumer` 可以有多個實例，`Partition` 分佈在多個 `Broker` 的話，`Consumer` 的多個實例就可以連接不同的 `Broker`，大大提升了消息處理能力。可以讓一個 `Consumer` 實例負責一個 `Partition`，這樣消息處理既清晰又高效。
- ## Partition 爲 Kafka 提供了數據冗餘
	- [[Kafka]] 爲一個 `Partition` 生成多個副本，並且把它們分散在不同的 `Broker`。
	  如果一個 `Broker` 故障了，`Consumer` 可以在其他 `Broker` 上找到 `Partition` 的副本，繼續獲取消息。
- ## 寫入 Partition
  一個 Topic 有多個 Partition，那麼，向一個 Topic 中發送消息的時候，具體是寫入哪個 Partition 呢？有 3 種寫入方式。
	- ### 使用 Partition Key 寫入特定 Partition
		- ![image.png](../assets/image_1726932179698_0.png)