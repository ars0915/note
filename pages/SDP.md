# Session-Level
	- `o=- 4611731400430051336 2 IN IP4 127.0.0.1`: session id, session version, network type
	  session version 在 SDP negotiation 時都會 +1
	- `s=-`: session name (often unused)
	- `t=0 0`: start and end time (often unused)
- # Group Attributes
	- `a=group:BUNDLE audio video`: The BUNDLE mechanism allows multiple media streams to share the same network connection (port and transport), which reduces resource usage.
	- `*a=msid-semantic: WMS lgsCFqt9kN2fVKw5wg3NKqGdATQoltEwOdMS*`: 
	  表示這個 SDP 用 Media Stream Identification (MSID) semantic