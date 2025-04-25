public:: true

- ## Caps
	- ### Set Caps on udpsrc
		- ```
		  udpsrc caps="application/x-rtp, media=video, encoding-name=H264, payload=96, clock-rate=90000" ! ...
		  ```
		  	•	The output pad of udpsrc will produce buffers with these caps.
		  	•	These caps are not a filter, they are an assertion: “This is the type of data I will emit.”
	- ### Filter with Caps on a Link
		- You add a caps filter as an element between stages:
		  
		  ```
		  ```