public:: true
tags:: Video Compression

- # Encode a video
  
  ```shell
  ffmpeg -i Documents/chrome.mp4 -c:v libx264 -tune animation  Documents/output.mkv
  ```
- # Plot Frame Type and Size
  
  ```shell
  plotframes -I output.mkv -t qt
  ```
  ![image.png](../assets/image_1733970262582_0.png)
- # Display Frame Information
  
  ```shell
  ffprobe -v trace -show_frames output.mkv
  ```
  output snippet
  ```shell
  [FRAME]
  media_type=video
  stream_index=0
  key_frame=1
  pts=3
  pts_time=0.003000
  pkt_dts=3
  pkt_dts_time=0.003000
  best_effort_timestamp=3
  best_effort_timestamp_time=0.003000
  duration=33
  duration_time=0.033000
  pkt_pos=4989
  pkt_size=21382
  width=480
  height=270
  crop_top=0
  crop_bottom=0
  crop_left=0
  crop_right=0
  pix_fmt=yuv420p
  sample_aspect_ratio=1:1
  pict_type=I
  interlaced_frame=0
  top_field_first=0
  repeat_pict=0
  color_range=tv
  color_space=unknown
  color_primaries=unknown
  color_transfer=unknown
  chroma_location=left
  [/FRAME]
  [FRAME]
  media_type=audio
  stream_index=1
  key_frame=1
  pts=3
  pts_time=0.003000
  pkt_dts=3
  pkt_dts_time=0.003000
  best_effort_timestamp=3
  best_effort_timestamp_time=0.003000
  duration=13
  duration_time=0.013000
  pkt_pos=27203
  pkt_size=1
  sample_fmt=fltp
  nb_samples=576
  channels=2
  channel_layout=stereo
  [/FRAME]
  ```
	- ## Frame type
		- When video key_frame = 1, pict_type = I
		  
		  ```shell
		  media_type=video
		  key_frame=1
		  pict_type=I
		  ```
	- ## Frame size
		- Mostly I frames is bigger than P and B frames
		  
		  ```shell
		  [FRAME]
		  pkt_size=4714
		  pict_type=I
		  [/FRAME]
		  
		  [FRAME]
		  pkt_size=123
		  pict_type=B
		  [/FRAME]
		  
		  [FRAME]
		  pkt_size=1610
		  pict_type=P
		  [/FRAME]
		  ```
	- ## Presentation and decode timestamps
		- `pts`: The presentation timestamp of the frame, often used in place of pkt_pts when analyzing frames.
		  `pkt_dts`: The decoding timestamp of the packet, indicating when the frame should be decoded.
		  `best_effort_timestamp`: FFmpeg’s best guess at the frame’s timestamp, derived from either pts, pkt_pts, or other available metadata.
		  `duration`: duration=33 and duration_time=0.033000 mean that this frame is displayed for 0.033 seconds (or 33 milliseconds) before the next frame is presented.
		  ```shell
		  [FRAME]
		  media_type=video
		  stream_index=0
		  key_frame=0
		  pts=26296
		  pts_time=26.296000
		  pkt_dts=26296
		  pkt_dts_time=26.296000
		  best_effort_timestamp=26296
		  best_effort_timestamp_time=26.296000
		  duration=33
		  duration_time=0.033000
		  [/FRAME]
		  ```
- # Inspect NAL Unit and Slice Information
  Use `ffmpeg` to trace NAL unit and slice-level details
  ```shell
  ffmpeg -i file.h264 -c copy -bsf:v trace_headers -f null -
  ```
	- `bsf:v trace_headers`:
	  Applies the trace_headers [bitstream](((675a7bdf-c641-4c0c-ba7d-421309ee1dc2))) filter to the video stream. This filter analyzes the H.264 bitstream and outputs information about headers in the stream, such as Sequence Parameter Sets (SPS), Picture Parameter Sets (PPS), and slice headers.
	- output snippet
		- SPS
		  
		  ```shell
		  ```
			- `forbidden_zero_bit`: Must always be 0. Ensures stream integrity.
			- `nal_ref_idc:` Indicates the importance of this NAL unit for decoding (3 = highest priority).
			- `nal_unit_type`: Specifies the type of NAL unit (7 = SPS).
			- `profile_idc`: Encoding profile used (100 = High Profile).
			- `level_idc`: Defines the video level (complexity) (21 = Level 2.1).
			- `chroma_format_idc`: Chroma subsampling format (1 = 4:2:0).
			- `bit_depth_luma_minus8` and `bit_depth_chroma_minus8`: Bit depth for luma and chroma (8-bit in this case).
			- `log2_max_frame_num_minus4`: Specifies the maximum number of frames between keyframes.
			- `pic_width_in_mbs_minus1`: Encoded width in macroblocks minus 1 (30 × 16 = 480 pixels wide).
			- `pic_height_in_map_units_minus1`: Encoded height in macroblock units minus 1 (17 × 16 = 272 pixels high).
			- `frame_mbs_only_flag`: Indicates whether only progressive frames are used (1 = yes).
			- `vui_parameters_present_flag`: Indicates if additional VUI (Video Usability Information) is present.
			- `aspect_ratio_idc`: Aspect ratio (1 = square pixels).
			- `time_scale` and `num_units_in_tick`: Timing information for frame rate. Here, 60000 / 1001 = ~59.94 fps.
		- PPS
		  ```shell
		  [trace_headers @ 0x600000e18140] Picture Parameter Set
		  [trace_headers @ 0x600000e18140] 3           nal_unit_type                                           01000 = 8
		  [trace_headers @ 0x600000e18140] 8           pic_parameter_set_id                                        1 = 0
		  [trace_headers @ 0x600000e18140] 9           seq_parameter_set_id                                        1 = 0
		  [trace_headers @ 0x600000e18140] 10          entropy_coding_mode_flag                                    1 = 1
		  [trace_headers @ 0x600000e18140] 13          num_ref_idx_l0_default_active_minus1                    00110 = 5
		  [trace_headers @ 0x600000e18140] 18          num_ref_idx_l1_default_active_minus1                        1 = 0
		  [trace_headers @ 0x600000e18140] 22          pic_init_qp_minus26                                     00111 = -3
		  [trace_headers @ 0x600000e18140] 33          deblocking_filter_control_present_flag                      1 = 1
		  ```
			- `nal_unit_type`: Specifies the type of NAL unit (8 = PPS).
			- `pic_parameter_set_id`: Identifier for this PPS.
			- `seq_parameter_set_id`: Links this PPS to its corresponding SPS.
			- `entropy_coding_mode_flag`: Specifies the entropy coding method (1 = CABAC, a more efficient method).
			- `num_ref_idx_l0_default_active_minus1`: Default number of reference frames for prediction.
			- `pic_init_qp_minus26`: Initial quantization parameter for the picture.
			- `deblocking_filter_control_present_flag`: Indicates if deblocking filter parameters are present.
		- Slice header
		  
		  ```shell
		  [trace_headers @ 0x600000e18140] Slice Header
		  [trace_headers @ 0x600000e18140] 0           forbidden_zero_bit                                          0 = 0
		  [trace_headers @ 0x600000e18140] 1           nal_ref_idc                                                10 = 2
		  [trace_headers @ 0x600000e18140] 3           nal_unit_type                                           00001 = 1
		  [trace_headers @ 0x600000e18140] 8           first_mb_in_slice                                           1 = 0
		  [trace_headers @ 0x600000e18140] 9           slice_type                                              00111 = 6
		  [trace_headers @ 0x600000e18140] 14          pic_parameter_set_id                                        1 = 0
		  [trace_headers @ 0x600000e18140] 15          frame_num                                                0010 = 2
		  [trace_headers @ 0x600000e18140] 19          pic_order_cnt_lsb                                      000100 = 4
		  [trace_headers @ 0x600000e18140] 25          direct_spatial_mv_pred_flag                                 1 = 1
		  [trace_headers @ 0x600000e18140] 26          num_ref_idx_active_override_flag                            1 = 1
		  [trace_headers @ 0x600000e18140] 27          num_ref_idx_l0_active_minus1                                1 = 0
		  [trace_headers @ 0x600000e18140] 28          num_ref_idx_l1_active_minus1                                1 = 0
		  [trace_headers @ 0x600000e18140] 29          ref_pic_list_modification_flag_l0                           0 = 0
		  [trace_headers @ 0x600000e18140] 30          ref_pic_list_modification_flag_l1                           0 = 0
		  [trace_headers @ 0x600000e18140] 31          adaptive_ref_pic_marking_mode_flag                          0 = 0
		  [trace_headers @ 0x600000e18140] 32          cabac_init_idc                                              1 = 0
		  [trace_headers @ 0x600000e18140] 33          slice_qp_delta                                          00101 = -2
		  [trace_headers @ 0x600000e18140] 38          disable_deblocking_filter_idc                               1 = 0
		  [trace_headers @ 0x600000e18140] 39          slice_alpha_c0_offset_div2                                010 = 1
		  [trace_headers @ 0x600000e18140] 42          slice_beta_offset_div2                                    010 = 1
		  [trace_headers @ 0x600000e18140] 45          cabac_alignment_one_bit                                     1 = 1
		  [trace_headers @ 0x600000e18140] 46          cabac_alignment_one_bit                                     1 = 1
		  [trace_headers @ 0x600000e18140] 47          cabac_alignment_one_bit                                     1 = 1
		  [trace_headers @ 0x600000e18140] Packet: 34 bytes, pts 36, dts 36, duration 33.
		  ```
	-