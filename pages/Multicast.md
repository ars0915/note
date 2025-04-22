public:: true

- ### **Valid Multicast IP Range**
  
  Multicast IPs are in the range:
  
  ```
  224.0.0.0 to 239.255.255.255
  ```
  
  So 224.5.5.5 is valid. However:
	- Addresses in 224.0.0.0 - 224.0.0.255 are **reserved** for local network control and shouldn’t be used for general-purpose multicasting.
-