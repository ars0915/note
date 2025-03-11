# Network
	- ## Port test
	  Tests if a TCP port is open and reachable
	  It's the most fundamental network connectivity check
	  pinpoint whether a failure is at the network/firewall level or at the application protocol level
		- ### Error
			- **CONNECTION_REFUSED**: The server actively refused the connection, which typically means the server is reachable but nothing is listening on that port
			- **TIMEOUT**: The connection timed out, which often indicates a firewall blocking the connection
			- **HOST_NOT_FOUND**: Host could not be resolved, indicating DNS or routing issues
			- **NO_ROUTE**: No route to the host, indicating network configuration issues
			- **CONNECTION_RESET**: Connection was reset, which could mean a firewall or server configured to drop connections
		-
	- ## Application connection test
	  These are higher-level protocols that run on top of TCP
	  They not only check port connectivity but also protocol-level communication
	  They verify that your specific server implementations are working
		-
	-