public:: true

- # video pipeline
	- udpsrc uri=udp://224.5.5.5:4002 multicast-iface=wlan0 caps="application/x-rtp, media=video, payload=96, clock-rate=90000, encoding-name=H264" ! rtpbin ! queue ! rtph264depay ! video/x-h264, stream-format=byte-stream , alignment=au ! appsink name=appsink sync=false
	- ## udpsrc
		- multicast-iface=wlan0: 指定 Wi-Fi interface
		- caps="application/x-rtp, ...": 告訴 GStreamer UDP data 是 RTP format、包含 H264 video
	- ## caps
		- ### Set Caps on udpsrc
			- ```
			  udpsrc caps="application/x-rtp, media=video, encoding-name=H264, payload=96, clock-rate=90000" ! ...
			  ```
			  	•	The output pad of udpsrc will produce buffers with these caps.
			  	•	These caps are not a filter, they are an assertion: “This is the type of data I will emit.”
		- ### Filter with Caps on a Link
			- You add a caps filter as an element between stages:
			  
			  ```
			  ... ! rtph264depay ! video/x-h264, stream-format=byte-stream, alignment=au ! appsink ...
			  ```
			  	•	The output of rtph264depay must match these caps.
			  	•	If it doesn’t, negotiation fails (or it tries to convert via other elements if autoplugging is enabled).
			  	•	This is how you force a certain format going into appsink.
			- 和在程式裡加一樣
			  
			  ```cpp
			  GstCaps* sink_caps = gst_caps_new_simple("video/x-h264",
			      "stream-format", G_TYPE_STRING, "byte-stream",
			      "alignment", G_TYPE_STRING, "au",
			      NULL);
			  g_object_set(appsink, "caps", sink_caps, NULL);
			  gst_caps_unref(sink_caps);
			  ```