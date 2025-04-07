public:: true

- # Glossary
	- ## Bitstream
	  id:: 675a7bdf-c641-4c0c-ba7d-421309ee1dc2
	  A bitstream is a sequence of binary data (bits) used to encode video or audio in a compressed format. It is essentially the output of an encoder and the input to a decoder. In video compression (e.g., H.264), the bitstream contains:
	  •	Encoded video data.
	  •	Metadata required to decode and display the video correctly.
	  The bitstream is structured in a way that allows efficient storage, transmission, and decoding while maintaining as much quality as possible within a given bitrate.
		- ### Headers in the Bitstream?
		  In the context of video compression, headers are sections of the bitstream that contain metadata describing how the encoded video data is structured and how it should be decoded. Headers play a critical role in ensuring that a decoder can interpret the bitstream correctly.
			- **Global Headers:**
			  These apply to the entire video stream and provide information necessary for decoding the video as a whole.
				- `SPS (Sequence Parameter Set)`: Defines global parameters like resolution, frame rate, bit depth, and color format.
				- `PPS (Picture Parameter Set)`: Contains parameters specific to decoding frames, such as entropy coding and reference frame settings.
			- **Frame Headers:**
			  These appear at the beginning of each frame and describe frame-specific details, like whether the frame is a keyframe or a predicted frame.
				- `Slice Header`: Contains information about a slice (portion of a frame) and specifies how it relates to other slices or frames.
			- **NAL Unit Headers (H.264/AVC Specific):**
			  The bitstream is divided into NAL Units (Network Abstraction Layer Units), each with a header that indicates the type of data it contains.
				- SPS, PPS, IDR frame (keyframe), non-IDR frame, etc.
				- The NAL Unit Header contains fields like:
					- `nal_unit_type`: Specifies the type of data (e.g., 5 for IDR frames, 7 for SPS, 8 for PPS).
					- `nal_ref_idc`: Indicates the importance of the NAL unit for decoding.
		- Example: H.264 Bitstream Components
		  In an H.264 bitstream, the structure might look like this:
		  ```shell
		  [Header: SPS] [Header: PPS] [Header: IDR Frame] [Slice Data] ...
		  ```
			- **SPS (Sequence Parameter Set):**
			  Defines global settings like:
				- Resolution: 1920x1080
				- Profile: High
				- Level: 4.1
			- **PPS (Picture Parameter Set):**
			  Frame-specific settings like:
				- Entropy coding method: CABAC or CAVLC
				- Number of reference frames.
			- **NAL Unit Headers:**
			  Contain fields like:
				- Type: SPS (7), PPS (8), IDR frame (5), or non-IDR frame (1).
	- ## Frame
		- I frame: Intraframe, Keyframe，Spatial
			- 不需參考其他幀即可編碼，參考同一張 frame 的附近 block
		- P frame: Predicted
			- 需要之前的 I frame 或 P frame 參考，只有變化的部份需要編碼
		- B frame: Bi-directional Predicted
			- 需要之前和之後的 I, P frame 編碼，缺點：延遲、live stream 要等待、運算量大
		- GOP: Group of Pictures
			- 代表 I frame 間的間隔，愈長檔案愈小，但畫質可能愈差
		-
		-
		-