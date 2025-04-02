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
		- 如果此时 WebRTC 扮演的角色为 `ICEROLE_CONTROLLED`，则对比 a 和 b 的远端提名次数，选择次数较高的那个；或者选择两者中最近收到过数据包的那个。
		- 如果前两步都没有对比出结果，则直接调用 `CompareConnectionCandidates` 进行对比。
		- 在连接过程中，总有一方是 Controlling，另一方是 Controlled。由 Controlling 负责决定最终选择哪个 ICE candidate 进行配对，并通过 STUN 协议告知 Controlled。具体可参见 [RFC 5245](https://datatracker.ietf.org/doc/html/rfc5245#section-3)。
			- Controlling Agent:  The ICE agent that is responsible for selecting
			      the final choice of candidate pairs and signaling them through
			      STUN and an updated offer, if needed.  In any session, one agent
			      is always controlling.  The other is the controlled agent.
			- Controlled Agent:  An ICE agent that waits for the controlling agent
			      to select the final choice of candidate pairs.
	- ## CompareConnectionStates
		- ```cpp
		  bool BasicIceController::PresumedWritable(const Connection* conn) const {
		    return (conn->write_state() == Connection::STATE_WRITE_INIT &&
		            config_.presume_writable_when_fully_relayed &&
		            conn->local_candidate().type() == RELAY_PORT_TYPE &&
		            (conn->remote_candidate().type() == RELAY_PORT_TYPE ||
		             conn->remote_candidate().type() == PRFLX_PORT_TYPE));
		  }
		  
		  // Compare two connections based on their writing, receiving, and connected
		  // states.
		  int BasicIceController::CompareConnectionStates(
		      const Connection* a,
		      const Connection* b,
		      absl::optional<int64_t> receiving_unchanged_threshold,
		      bool* missed_receiving_unchanged_threshold) const {
		    // First, prefer a connection that's writable or presumed writable over
		    // one that's not writable.
		    bool a_writable = a->writable() || PresumedWritable(a);
		    bool b_writable = b->writable() || PresumedWritable(b);
		    if (a_writable && !b_writable) {
		      return a_is_better;
		    }
		    if (!a_writable && b_writable) {
		      return b_is_better;
		    }
		  
		    // Sort based on write-state. Better states have lower values.
		    if (a->write_state() < b->write_state()) {
		      return a_is_better;
		    }
		    if (b->write_state() < a->write_state()) {
		      return b_is_better;
		    }
		  
		    // We prefer a receiving connection to a non-receiving, higher-priority
		    // connection when sorting connections and choosing which connection to
		    // switch to.
		    if (a->receiving() && !b->receiving()) {
		      return a_is_better;
		    }
		    if (!a->receiving() && b->receiving()) {
		      if (!receiving_unchanged_threshold ||
		          (a->receiving_unchanged_since() <= *receiving_unchanged_threshold &&
		           b->receiving_unchanged_since() <= *receiving_unchanged_threshold)) {
		        return b_is_better;
		      }
		      *missed_receiving_unchanged_threshold = true;
		    }
		  
		    // some comments...
		  
		    // In the case where we reconnect TCP connections, the original best
		    // connection is disconnected without changing to WRITE_TIMEOUT. In this case,
		    // the new connection, when it becomes writable, should have higher priority.
		    if (a->write_state() == Connection::STATE_WRITABLE &&
		        b->write_state() == Connection::STATE_WRITABLE) {
		      if (a->connected() && !b->connected()) {
		        return a_is_better;
		      }
		      if (!a->connected() && b->connected()) {
		        return b_is_better;
		      }
		    }
		  
		    return 0;
		  }
		  ```
		- 对于 a 和 b，首先选择两者中可写入（writable）或者假定（presume）可写入的那个。
		  假定可写入的判断条件可以简单认为是 local candidate 为 relay 模式，且 remote candidate 为 relay 或者 [[prflx]] 模式。
		  relay 模式即为 TURN 模式； [[prflx]] 模式指的是在使用 STUN 模式连接成功后，WebRTC 发现可以与远端直连了，便会本地生成新的 remote candidate 作为候选。
		- 如果前一田山中选择 write_state 更小的那个。
		  Connection::WriteState 的枚举定义如下：
			- `STATE_WRITABLE = 0` 表示近期有收到过 ping responses；
			  `STATE_WRITE_UNRELIABLE = 1` 表示有一些 ping 发送失败了；
			  `STATE_WRITE_INIT = 2` 表示还没有受到过 ping response；
			  `STATE_WRITE_TIMEOUT = 3` 表示有大量 ping 发送失败，可以认为是连接超时。
		- 或者选择两者中正在接收数据（receiving）的那个，判断是否 receiving 可以参考 `Connection::UpdateReceiving`。
		  注意 a 和 b 存在顺序关系，如果要切换到 b 还需要满足一定的阈值（threshold）；不过目前源码中 SortAndSwitchConnection 调用 CompareConnections 时并没有设置阈值，所以可以直接切换。
		- 或者当 a 和 b 都是 TCP 连接，且两者的 `write_state` 都为 `STATE_WRITABLE` ，选择两者中已经连接成功（connected）的那个。設定 connected 的逻辑可以参考 `Connection::set_connected`。
			- 当 TCP 断连时，主动方（active）会尝试重连 5s，期间仍然保持原连接 writable 状态不变。被动方（passive）也会保持原连接 writable 状态不变；且重连成功时会创建一条新连接，当新连接变为 writable 状态时，显然应该选择它。
		- 如果以上条件均不满足，则认为 a 和 b 相等。
		-
		-