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
	- Constructor with Only NetworkManager
	  ```cpp
	  explicit BasicPortAllocator(rtc::NetworkManager* network_manager);
	  ```