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
		- 调用 `absl::c_stable_sort`排序；当两个元素相等时，`absl::c_stable_sort` 将保证这两个元素之间的顺序关系。
		- 排序规则主要由 `CompareConnections` 实现；如果 `CompareConnections` 判断 a 和 b 相等，则两者中 RTT（Round-Trip Time）较小的那个将排在前面。
		- 排序完毕后，还需要调用 `ShouldSwitchConnection` 确认是否真的需要切换到新连接。
	- ## CompareConnections
		- ```cpp
		  int BasicIceController::CompareConnections(
		      const Connection* a,
		      const Connection* b,
		      absl::optional<int64_t> receiving_unchanged_threshold,
		      bool* missed_receiving_unchanged_threshold) const {
		    RTC_CHECK(a != nullptr);
		    RTC_CHECK(b != nullptr);
		  
		    // We prefer to switch to a writable and receiving connection over a
		    // non-writable or non-receiving connection, even if the latter has
		    // been nominated by the controlling side.
		    int state_cmp = CompareConnectionStates(a, b, receiving_unchanged_threshold,
		                                            missed_receiving_unchanged_threshold);
		    if (state_cmp != 0) {
		      return state_cmp;
		    }
		  
		    if (ice_role_func_() == ICEROLE_CONTROLLED) {
		      // Compare the connections based on the nomination states and the last data
		      // received time if this is on the controlled side.
		      if (a->remote_nomination() > b->remote_nomination()) {
		        return a_is_better;
		      }
		      if (a->remote_nomination() < b->remote_nomination()) {
		        return b_is_better;
		      }
		  
		      if (a->last_data_received() > b->last_data_received()) {
		        return a_is_better;
		      }
		      if (a->last_data_received() < b->last_data_received()) {
		        return b_is_better;
		      }
		    }
		  
		    // Compare the network cost and priority.
		    return CompareConnectionCandidates(a, b);
		  }
		  ```
		- 调用 `CompareConnectionStates` 对比 a 和 b，如果 `state_cmp != 0`（两者不相等），则直接将 `state_cmp` 作为结果返回。
		- 如果此时 WebRTC 扮演的角色为 `ICEROLE_CONTROLLED`（被控制者），则对比 a 和 b 的远端提名次数，选择次数较高的那个；或者选择两者中最近收到过数据包的那个。
		- 如果前两步都没有对比出结果，则直接调用 `CompareConnectionCandidates` 进行对比。
		- 在连接过程中，总有一方是控制者（controlling），另一方是被控制者（controlled）。由控制者负责决定最终选择哪个 ICE candidate 进行配对，并通过 STUN 协议告知被控制者。具体可参见 [RFC 5245](https://datatracker.ietf.org/doc/html/rfc5245#section-3)。
			-
		-