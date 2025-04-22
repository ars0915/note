public:: true

- ### **✅ **
- ### **Valid Multicast IP Range**
  
  Multicast IPs are in the range:
  
  ```
  224.0.0.0 to 239.255.255.255
  ```
  
  So 224.5.5.5 is valid. However:
- Addresses in 224.0.0.0 - 224.0.0.255 are **reserved** for local network control and shouldn’t be used for general-purpose multicasting.
- Others, like 224.5.5.5, are suitable for application-specific multicast.
  
  ---
- ### **⚠️ **
- ### **“use of closed network connection” Error**
  
  This error typically means your socket is being closed unexpectedly. Here are a few potential causes:
- **The socket was explicitly closed before the send call.**
- **Another part of your code (or OS event) closed the connection.**
- **An exception or panic might have closed it before sending.**
  
  This error is **not** multicast-specific — it’s about the socket’s lifecycle.
  
  ---
- ### **🔧 **
- ### **Checklist to Multicast Properly in Go (Example)**
  
  Here’s a minimal example of how to send and receive a multicast packet in Go:
- #### **Sending (Multicast):**
  
  ```
  addr, _ := net.ResolveUDPAddr("udp", "224.5.5.5:4000")
  conn, err := net.DialUDP("udp", nil, addr)
  if err != nil {
    log.Fatal(err)
  }
  defer conn.Close()
  
  _, err = conn.Write([]byte("Hello multicast"))
  if err != nil {
    log.Fatal(err)
  }
  ```
- #### **Receiving (Joining Multicast Group):**
  
  ```
  group := net.ParseIP("224.5.5.5")
  ifi, _ := net.InterfaceByName("eth0") // or "en0", "wlan0", etc.
  
  addr := &net.UDPAddr{
    IP:   group,
    Port: 4000,
  }
  
  conn, err := net.ListenMulticastUDP("udp", ifi, addr)
  if err != nil {
    log.Fatal(err)
  }
  defer conn.Close()
  
  conn.SetReadBuffer(1024)
  buf := make([]byte, 1024)
  n, src, err := conn.ReadFromUDP(buf)
  fmt.Printf("Received %s from %s\n", buf[:n], src)
  ```
  
  ---
- ### **🧠 Key Points**
- Make sure the **interface name** (eth0, wlan0, etc.) is correct on your system.
- Ensure **firewall settings** or **network restrictions** aren’t blocking multicast.
- **Don’t close the socket** before sending the packet.
  
  ---
  
  If you’re still getting the “closed network connection” error, feel free to paste the relevant part of your code here — I can help debug it specifically.