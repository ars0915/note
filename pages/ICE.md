public:: true

- # 連接排序規則
	- 当 Peer Initiator 收到 answerSdp 之后便会开始 ICE 流程。在 answerSdp 中可能包含多条 ICE candidates（候选服务器）信息，此时 WebRTC 便会分别和这些 candidates 建立连接，然后选出其中最优的那条连接作为配对结果进行通话。
	- ## SortAndSwitchConnection
		- ```cpp
		  IceControllerInterface::SwitchResult
		  BasicIceController::SortAndSwitchConnection(IceControllerEvent reason) {
		    // Find the best alternative connection by sorting.  It is important to note
		    // that amongst equal preference, writable connections, this will choose the
		    // one whose estimated latency is lowest.  So it is the only one that we
		    // need to consider switching to.
		    // TODO(honghaiz): Don't sort;  Just use std::max_element in the right places.
		    absl::c_stable_sort(
		        connections_, [this](const Connection* a, const Connection* b) {
		          int cmp = CompareConnections(a, b, absl::nullopt, nullptr);
		          if (cmp != 0) {
		            return cmp > 0;
		          }
		          // Otherwise, sort based on latency estimate.
		          return a->rtt() < b->rtt();
		        });
		  
		    // method body...
		  
		    const Connection* top_connection =
		        (!connections_.empty()) ? connections_[0] : nullptr;
		  
		    return ShouldSwitchConnection(reason, top_connection);
		  }
		  ```
		- 调用 absl::c_stable_sort排序；当两个元素相等时，absl::c_stable_sort 将保证这两个元素之间的顺序关系。
		- 排序规则主要由 CompareConnections 实现；如果 CompareConnections 判断 a 和 b 相等，则两者中 RTT（Round-Trip Time）较小的那个将排在前面。
		  排序完毕后，还需要调用 ShouldSwitchConnection 确认是否真的需要切换到新连接。
		-