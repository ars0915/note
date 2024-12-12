# Encode a video

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
  ```
	- ## Frame type
		- When video key_frame = 1, pict_type = I
		  
		  ```shell
		  media_type=video
		  key_frame=1
		  pict_type=I
		  ```
	- ## Frame size
	- ## Presentation and decode timestamps