---
layout: post
title:  "Fooling around with ABS, HLS, and FFmpeg"
date:   2024-07-01 23:34:26 -0400
categories: abs hls ffmpeg noodling
excerpt_separator: <!--more-->
---

I've been messing around with ABS a bit and thought I'd throw up this little project in case anyone else wanted to mess around with it too. I learned a lot more about the [FFmpeg](https://ffmpeg.org) command line tool, ABS, HMS, and other random bits throughout the process.
<!--more-->

## Prerequsites 
First, capture or snip a little video that you can quickly generate HLS segements and playlists for different resolutions. For example, I used a 8 sec video of my little lady swinging in Tiburon, but not to plaster her adorable face across the internet, I've opted to use the Blue Angles from SF's Fleet Week 2023.

Also, if you're on a Mac and are using a video that is from your Photos library, my snippets below assume you're using a `.mp4` and not a `.MOV`. You can use FFmpeg to perform the conversion pretty easily. 

```bash
  ffmpeg -i input.mov -c:v libx264 -c:a aac -movflags +faststart input.mp4
```

## Steps 

### Transcode it
Use the following FFmpeg command to generate different resolutions of segments:
```bash
ffmpeg -i input.mp4 -filter_complex \
  "[0:v]scale=trunc(oh*a/2)*2:720[v720]; \
   [0:v]scale=trunc(oh*a/2)*2:480[v480]; \
   [0:v]scale=trunc(oh*a/2)*2:360[v360]" \
  -map "[v720]" -map a -c:v:0 h264 -b:v:0 2000k -maxrate:v:0 2000k -bufsize:v:0 4000k -hls_time 4 -hls_playlist_type vod -hls_segment_filename "720p_%03d.ts" 720p.m3u8 \
  -map "[v480]" -map a -c:v:1 h264 -b:v:1 1000k -maxrate:v:1 1000k -bufsize:v:1 2000k -hls_time 4 -hls_playlist_type vod -hls_segment_filename "480p_%03d.ts" 480p.m3u8 \
  -map "[v360]" -map a -c:v:2 h264 -b:v:2 500k -maxrate:v:2 500k -bufsize:v:2 1000k -hls_time 4 -hls_playlist_type vod -hls_segment_filename "360p_%03d.ts" 360p.m3u8
```

_Note: I used `oh*a/` inside FFmpeg's scale filter to calculate the width to maintain the aspect ratio while also keeping it even. Video codecs typically require width (and sometimes height) to be even. From what I understand, it's because of how video data is stored + processed in blocks of pixels._

### er.. Platlist it

ABR allows the video player to switch between different quality levels (bitrate and resolution) based on the viewer's network conditions. This is the basis for how Netflix and Disney+ ensure a smooth viewing experience with the least amount of buffering and drops in video quality when streaming on device.

```bash
echo "#EXTM3U
#EXT-X-STREAM-INF:BANDWIDTH=2000000,RESOLUTION=1280x720
720p.m3u8
#EXT-X-STREAM-INF:BANDWIDTH=1000000,RESOLUTION=854x480
480p.m3u8
#EXT-X-STREAM-INF:BANDWIDTH=500000,RESOLUTION=640x360
360p.m3u8" > master.m3u8
```
There's a ton to look at in FFmpeg's docs [here](https://ffmpeg.org/ffmpeg-formats.html#hls-2).


### Serve it

Next, we'll want to play the video on a webpage someplace. We can use [Video.js](https://videojs.com/guides/embeds/)'s player (popular video player library) to handle the HLS playback. It's a fantastic library that I've used in the past, and required seemingly little config.

You can then fire up a server to see it in action. Python's SimpleServer did the trick:

```python
python -m http.server 8000
```

### So what happens?

<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>HLS Streaming</title>
    <link href="https://vjs.zencdn.net/7.11.4/video-js.css" rel="stylesheet" />
    <script src="https://vjs.zencdn.net/7.11.4/video.js"></script>
</head>
<body>
    <video id="my-video" class="video-js vjs-default-skin" controls preload="auto" width="640" height="360"
           data-setup='{}'>
        <source src="{{ site.baseurl }}/assets/videos/master.m3u8" type="application/x-mpegURL">
    </video>
</body>
</html>

- When the video player starts, it loads `master.m3u8`.
- It then determines the viewer’s (your) current bandwidth and device capabilities.
- Based on the conditions, the player selects the best quality stream (e.g., 720p, 480p, or 360p).
- If network conditions change, the player can switch to a different variant stream seamlessly. Try playing around with the Network dev tools to see how the player dynamically switches to a different stream vairant. 

That's it–enjoy!