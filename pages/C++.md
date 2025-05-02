public:: true

- ## Constructors
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
			- A **std::unique_ptr is a move-only object**, meaning it cannot be copied. Attempting to use it without a reference (auto session) would result in a compilation error.
			- Using const auto& correctly references the std::unique_ptr without trying to copy it.
		- ### Why Not Use PortAllocatorSession Directly?
			- Direct Object Access:
			  You can’t directly iterate using PortAllocatorSession because the vector doesn’t store PortAllocatorSession objects. It stores std::unique_ptr<PortAllocatorSession>.
			  ```cpp
			  for (const PortAllocatorSession& session : allocator_sessions_) {  // ❌ ERROR
			  }
			  ```
			- Dereferencing (More Verbose):
			  If you explicitly want to work with the underlying object, you could write:
			  ```cpp
			  for (const auto& session_ptr : allocator_sessions_) {
			    const PortAllocatorSession& session = *session_ptr;
			    if (session.IsStopped()) {
			      continue;
			    }
			    session_ptr->StopGettingPorts();
			  }
			  ```
- ## sigslot
	- ```cpp
	  // PeerConnection 继承自 sigslot::has_slots
	  bool PeerConnection::Initialize(/* args... */) {
	    // method body...
	  
	    // std::unique_ptr<JsepTransportController> transport_controller_;
	    transport_controller_.reset(new JsepTransportController(/* args... */));
	    // 调用 connect 时需要传入两个参数，即接收回调的对象的指针，和回调方法的指针
	    transport_controller_->SignalIceConnectionState.connect(
	        this, &PeerConnection::OnTransportControllerConnectionState);
	  
	    // method body...
	  }
	  ```
	  ```cpp
	  class JsepTransportController : /* extends... */ {
	   public:
	    // other definitions...
	  
	    // sigslot::signal 支持操作符重载，
	    // void operator()(Args... args) { emit(args...); }
	    sigslot::signal1<cricket::IceConnectionState> SignalIceConnectionState;
	  
	    // other definitions...
	  }
	  ```
	  当 JsepTransportController 调用 `SignalIceConnectionState(state)` 时，便会回调 `PeerConnection::OnTransportControllerConnectionState` 这个方法；事实上只要 connect 了的方法，都会被回调。对于 Java(Script) 开发者来说，这种回调机制肯定不陌生，其实就是一种观察者模式的实现，**主要是为了代码解耦。**
- ## 左值 右值
	- ### 左值 Lvalue
	  出現在賦值運算子左邊的值，具名且可定位
		- 有名字、記憶體位址、可被取值(&)、可賦新值
		  
		  ```cpp
		  int x = 10;
		  x = 20;
		  int* p = &x;
		  ```
	- ### 右值 Rvalue
	  臨時值、沒有名字、無法持久存在
	  
	  ```cpp
	  int y = x + 1; // x+ 1 是右值
	  int z = 42; // 42 是右值
	  ```
	- ### std::move 把左值轉為右值參考 (rvalue reference)，讓它可以被移動
		- ```cpp
		  std::string a = "hello"; // a 是左值
		  std::string b = std::move(a); // ✅ 移動 a 的資源到 b, std::move(a)右值
		  ```
	- ### 左值引用、右值引用
		- 左值引用: T& 接收一般變數，具名物件
		- 右值引用: T&& 接收臨時物件，允許資源轉移
	- ### 為什麼要把左值轉成右值
		- 右值才能觸發「移動語意(move semantics)」資源轉移
		  避免「copy」，特別在處理大型資源時(std::vector, std::string, std::unique_ptr)
-
-
- # Reference
- "信號槽機制," *WebRTC 學習指南*, Available: [link_to_page](https://webrtc.mthli.com/code/sigslot/).