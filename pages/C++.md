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
		- 左值引用: `T&` 接收一般變數，具名物件
		- 右值引用: `T&&` 接收臨時物件，允許資源轉移
	- ### 為什麼要把左值轉成右值
		- 右值才能觸發「移動語意(move semantics)」資源轉移
		  避免「copy」，特別在處理大型資源時(std::vector, std::string, std::unique_ptr)
		- move 比 copy 快
		  
		  ```cpp
		  std::vector<int> v1 = {1, 2, 3, 4};
		  
		  // 這是拷貝：v2 是 v1 的副本
		  std::vector<int> v2 = v1;
		  
		  // 這是移動：v3 直接接手 v1 的資源，v1 變空
		  std::vector<int> v3 = std::move(v1);
		  ```
	- ### 為什麼需要 std::move
		- 傳遞 move-only 型別（如 std::unique_ptr）
		  
		  ```cpp
		  void take(std::unique_ptr<int> p); // 只接受右值
		  
		  std::unique_ptr<int> up = std::make_unique<int>(5);
		  take(std::move(up));  // ✅ 必須轉成右值才可移動
		  ```
			- std::unique_ptr<T> 是一種 獨佔式智能指標，保證 只會有一個擁有者。
			  它 不能被複製（copy），只能被移動（move）。
			  
			  這樣合法（傳右值）
			  ```cpp
			  std::unique_ptr<int> up = std::make_unique<int>(5);
			  take(std::move(up)); // ✅ 移動給函數
			  ```
			  這樣錯誤（傳左值）
			  ```cpp
			  std::unique_ptr<int> up = std::make_unique<int>(5);
			  take(up); // ❌ 編譯錯誤，因為會試圖複製 up
			  ```
		- 避免拷貝大型物件
		  
		  ```cpp
		  std::string makeMessage() {
		      std::string msg = "Hello from a big string buffer...";
		      return std::move(msg); // ✅ 可顯式提示移動（雖然編譯器可優化）
		  }
		  ```
	- ### std::move 不是真的在「移動」
	  std::move(x) 只是把 x 的型別強制轉成右值參考（T&&），它不會真的做「移動」。
	  真正移動的是 constructor 或 assignment operator：
	  
	  ```cpp
	  std::string a = "hello";
	  std::string b = std::move(a); // b 會調用 std::string 的 move constructor
	  ```
- ##  absl::AnyInvocable
	- absl::AnyInvocable 是 Google Abseil C++ Library 提供的一個輕量版、功能更安全的函數封裝類別，類似於 std::function，但支援：
		- 僅限一次執行（move-only）
		- 支援 const、&、&& 調用（invocation qualifiers）
		- 可儲存 lambda、函數物件、綁定的 callback，包含 move-only lambda
	- 相比 std::function 必須可複製，absl::AnyInvocable 允許 move-only 的 lambda 或 callable，這讓它更適合 callback、任務封裝等用途。
	- ### absl::AnyInvocable<void() &&>
		- 表示它只接受一個可呼叫的物件，且只能透過右值調用（即 std::move(callback)();)
		  && (rvalue ref): 表示這個 callable **只能被右值調用**（例如只允許一次）
		- 用途場景:
		  單次 callback
		  
		  ```cpp
		  absl::AnyInvocable<void() &&> task = [ptr = std::make_unique<int>(42)]() mutable {
		      std::cout << "Value: " << *ptr << std::endl;
		  };
		  
		  std::move(task)(); // ✅ OK，一次性移動並執行
		  ```
-
-
- # Reference
- "信號槽機制," *WebRTC 學習指南*, Available: [link_to_page](https://webrtc.mthli.com/code/sigslot/).