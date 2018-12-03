

Text: https://trac.ffmpeg.org/wiki/Hardware/VAAPI

Original (10fps 250% CPU):
ffmpeg -rtsp_transport http -re -i rtsp://nvr:7447/a1b2c3_1?api_key=a1b2c3 -threads 0 -vcodec libx264 -an -pix_fmt yuv420p -r 30 -f rawvideo -tune zerolatency -vf scale=1920:1080 -payload_type 99 -ssrc 13686140 temp.mp4


Works (30fps 50% CPU):
ffmpeg -vaapi_device /dev/dri/renderD128 -rtsp_transport http -re -i rtsp://nvr:7447/a1b2c3_1?api_key=a1b2c3 -threads 0 -vcodec libx264 -an -pix_fmt yuv420p -r 30 -f rawvideo -tune zerolatency -vf 'scale=1920:1080,format=nv12,hwupload' -c:v h264_vaapi -payload_type 99 -ssrc 13686140 temp.mp4

Native Docker in Synology:
```json
   "devices" : [
      {
         "CgroupPermissions": "rwm",
         "PathInContainer": "\/dev/dri",
         "PathOnHost": "\/dev\/dri"
      }
   ],
```

I've tried and failed:
 * Building FFMPEG with --enable-libmfx
 * Using dockerized ffmpeg
 * Using synology's one (missing -hwaccels completely and 2.7)
 * h264_qsv (requires some intel magic to be installed)
