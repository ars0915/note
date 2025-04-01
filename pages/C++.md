## Constructors
	- ```cpp
	  class RTC_EXPORT BasicPortAllocator : public PortAllocator {
	   public:
	    // note: The (optional) relay_port_factory is owned by caller
	    // and must have a life time that exceeds that of BasicPortAllocator.
	    BasicPortAllocator(rtc::NetworkManager* network_manager,
	                       rtc::PacketSocketFactory* socket_factory,
	                       webrtc::TurnCustomizer* customizer = nullptr,
	                       RelayPortFactoryInterface* relay_port_factory = nullptr);
	    explicit BasicPortAllocator(rtc::NetworkManager* network_manager);
	    BasicPortAllocator(rtc::NetworkManager* network_manager,
	                       const ServerAddresses& stun_servers);
	    BasicPortAllocator(rtc::NetworkManager* network_manager,
	                       rtc::PacketSocketFactory* socket_factory,
	                       const ServerAddresses& stun_servers);
	  ```
	- There are multiple overloaded constructors, allowing instances of BasicPortAllocator to be initialized in different ways.
		- Constructor with Only NetworkManager
		  ```cpp
		  explicit BasicPortAllocator(rtc::NetworkManager* network_manager);
		  ```
		  避免在沒有意想到的地方發生隱式轉換 (implicit conversion)
		  例如
		  ```cpp
		  BasicPortAllocator a1 = network_manager; //隱式轉換會 compile error
		  BasicPortAllocator a2{network_manager}; 
		  ```
- ## Loop by pointer
	- ```cpp
	  std::vector<std::unique_ptr<webrtc::PortAllocatorSession>> allocator_sessions_;
	  for (const auto& session : allocator_sessions_) {
	  	if (session->IsStopped()) {
	      	continue;
	      }
	  	session->StopGettingPorts();
	  }
	  ```
		- const: 
		  The const ensures that session is read-only and cannot be modified within the loop. This is appropriate if you **don’t intend to modify the unique_ptr** itself, only call functions on the pointed-to object.
		- auto&:
			- auto& is a reference to the actual element in the vector, instead of making a copy of it.
			- Since allocator_sessions_ is a std::vector<std::unique_ptr<webrtc::PortAllocatorSession>>, each element is a std::unique_ptr<PortAllocatorSession>.
			- Using auto& ensures that the code is efficient by **avoiding unnecessary copies** of the smart pointer.
		- Using std::unique_ptr:
		  	•	A std::unique_ptr is a move-only object, meaning it cannot be copied. Attempting to use it without a reference (auto session) would result in a compilation error.
		  	•	Using const auto& correctly references the std::unique_ptr without trying to copy it.
		  	4.	Calling Methods via Pointer:
		  	•	session->IsStopped() and session->StopGettingPorts(); work because session is a std::unique_ptr, and the -> operator accesses the underlying PortAllocatorSession.