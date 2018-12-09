---
title:  "Hardware-Accelerated h264 Encoding on Synology NAS"
---

Many Synology NAS do have an Intel CPU that supports hardware-accelerated
h264 encoding, which Intel calls QuickSync for marketing purposes.
You would get around 10x improvement and most importantly real-time
video transcoding with low latency.
Surprisingly they seemingly do not use it themselves internally, but it's
possible to use it manually. This easily works from within Docker as well.

## Synology NAS Models with Hardware h264

These instructions were verified on
[Synology NAS DiskStation DS718+](https://www.synology.com/en-us/products/DS718+)
which uses
[Intel Celeron J3455](https://www.intel.com/content/www/us/en/products/processors/celeron/j3455.html).
It's the same CPU as DS918+, so this should apply to that model as
well. Similar
[Intel Celeron J3355](https://ark.intel.com/products/95597/Intel-Celeron-Processor-J3355-2M-Cache-up-to-2-5-GHz-)
also has QuickSync, so this would _probably_ apply to DS218+.

<!--more-->

DS418play has J3355 as well, but it does not officially support Docker.
You still can probably can make it work somehow.

CPUs that do not support QuickSync and do not support hardware acceleration:
  * [Intel Atom C3538](https://ark.intel.com/products/97929/Intel-Atom-Processor-C3538-8M-Cache-up-to-2-10-GHz-)
    and
    [Intel Atom C2538](https://ark.intel.com/products/77981/Intel-Atom-Processor-C2538-2M-Cache-2-40-GHz-)
    which are used for DS1517+, DS1618+, DS1817+, DS1819+, DS2415+,
    RS818+, RS1219+, RS2418+, RS2418RP+, RS2819RP+ Synology NAS models
  * [Intel Xeon D-1541](https://ark.intel.com/products/91199/Intel-Xeon-Processor-D-1541-12M-Cache-2-10-GHz-),
    [Intel Xeon D-1531](https://ark.intel.com/products/91203/Intel-Xeon-Processor-D-1531-9M-Cache-2-20-GHz-),
    [Intel Xeon D-1521](https://ark.intel.com/products/91202/Intel-Xeon-Processor-D-1521-6M-Cache-2-40-GHz-),
    [Intel Xeon D-1527](https://ark.intel.com/products/91195/Intel-Xeon-Processor-D-1527-6M-Cache-2-20-GHz-)
    which are used for for RS18017xs+, RS3618xs, RS4017xs+, DS3617xs,
    RS1619xs+, RS3617RPxs, RS3617xs+ Synology NAS models
  * [Intel Pentium D1508](https://ark.intel.com/products/91558/Intel-Pentium-Processor-D1508-3M-Cache-2-20-GHz-)
     which are used in FS1018, DS3018xs Synology NAS models

All other NASes from Synology as of 2018 use Realtek CPUs,
I do not know if they support it or not, but I lean heavily on a "no" side.


## Different Flavors of Hardware-Accelerated _ffmpeg_

_ffmpeg_ supports
[_many_ different types of accelerated encoding](https://trac.ffmpeg.org/wiki/HWAccelIntro).
Luckly for us only _libmfx_, _OpenCL_, and _VAAPI_ are supported
by Intel CPUs on Linux. _OpenCL_ implementation does not support hardware
encoding, and
[_libmfx_ is very hard to use on Linux](https://trac.ffmpeg.org/wiki/HWAccelIntro#libmfx)
which leaves us with only one possibility: _VAAPI_.

> If you see `h264_qsv` recommended somewhere it would use _libmfx_ under the
hood. I have not found a simple way to make it work.

### Synology's Own _ffmpeg_

I was surprised to find _ffmpeg_ version 2.7, which misses some of the hardware
acceleration implementations and was released back in 2015:
```sh
$ ffmpeg 2>&1 | head -n2
ffmpeg version 2.7.1 Copyright (c) 2000-2015 the FFmpeg developers
  built with gcc 4.9.3 (crosstool-NG 1.20.0) 20150311 (prerelease)
```

Which means it does not support any hardware implementations:
```sh
$ ffmpeg -buildconf 2>/dev/null | grep 'vaapi\|hw'
    --disable-vaapi
```

> What's weird is that `/dev/dri/*` devices are present and initialized,
which hints that Synology can somehow use hardware encoding.
Most likely I'm just looking into the wrong place.

### Debian's _ffmpeg_ and _VAAPI_

> Check [_VAAPI_ documentation](https://trac.ffmpeg.org/wiki/Hardware/VAAPI)
for all the internal details, I would only show a very short summary.

_VAAPI_ is a magical API that allows _ffmpeg_ to use hardware acceleration
for different video-related operations and works across different hardware.
It's always one of `/dev/dri/*` devices that can be used to talk to
the underlying hardware. We only need one for our purposes:
`/dev/dri/renderD128` (literally, `D128` is the same across platforms).

> I had some issues when I tried to do both h264 decoding _and_ encoding
at the same time, be careful with `-vf 'format=nv12'` and related parameters
and you can make it work. If you have a warning from _ffmpeg_ about
unstable threads, do not ignore it and try different options.

Options to add to enable _VAAPI_:
  * Use _VAAPI_ with a specific device `-vaapi_device /dev/dri/renderD128`
  * Make frame buffer format conversion to make hardware codec happy:
    `-vf 'format=nv12,hwupload'`. For example
    `-vf 'scale=1920:1080'` is replaced with
    `-vf 'scale=1920:1080,format=nv12,hwupload'`
    or `-vf 'scale_vaapi=w=1280:h=720'`
  * Actually use h264-codec with _VAAPI_ `-c:v h264_vaapi`

Simplest command to verify your encoding performance
using an example video [Big Buck Bunny](http://bbb3d.renderfarming.net):
```sh
sudo docker run --rm \
  --device /dev/dri:/dev/dri \
  jrottenberg/ffmpeg:vaapi \
  -vaapi_device /dev/dri/renderD128 \
  -i http://distribution.bbb3d.renderfarming.net/video/mp4/bbb_sunflower_1080p_30fps_normal.mp4 \
  /tmp/example.mp4
```

> Check a longer example in a performance section below.

## Synology Configs & Docker

Normally I'd do something like this:
```sh
docker run --device /dev/dri:/dev/dri jrottenberg/ffmpeg:vaapi ...
```

Synology's OS DSM 6.x uses its own configuration format for Docker and
does not easily allow one to override `docker run` command's command line
parameters. I have not found a documented way to configure it, but if you
configure a Docker container via a web UI and and "export" config into a file
you can add this into a plain JSON to configure devices mount:
```json
   "devices" : [
      {
         "CgroupPermissions": "rwm",
         "PathInContainer": "\/dev/dri",
         "PathOnHost": "\/dev\/dri"
      }
   ],
```

> It was very surprising, but yes, Synology does not document its internal
format for a Docker JSON. Literally no matches except for binaries:
```sh
```

There is no simple way of calling Dockerized _ffmpeg_ from
an another Docker image, but if you use Debian-based docker image chances
are it would be as easy as:
```sh
apt-get install ffmpeg
```

This may pull in the up-to-date version of _ffmpeg_ with all the right bindings
and devices.

## Transcoding Performance Results

### Big Buck Bunny: scaling from 1080p into 720p

| | fps | CPU% | fps/CPU core |
|-:|-|-|-|
| Software: | 30 | 380% | 8 |
| Hardware: | 110 | 85% | 130 |
| *Improvement:* | 3x | 5x | 15x |


Software transcoding example command:
```sh
sudo docker run --rm \
  --device /dev/dri:/dev/dri \
  jrottenberg/ffmpeg:vaapi \
  -i http://distribution.bbb3d.renderfarming.net/video/mp4/bbb_sunflower_1080p_30fps_normal.mp4 \
  -vf 'scale=1280:720' \
  /tmp/example.mp4
```

Hardware-accelerated transcoding example command:
```sh
sudo docker run --rm \
  --device /dev/dri:/dev/dri \
  jrottenberg/ffmpeg:vaapi \
  -hwaccel vaapi -vaapi_device /dev/dri/renderD128 -hwaccel_output_format vaapi \
  -i http://distribution.bbb3d.renderfarming.net/video/mp4/bbb_sunflower_1080p_30fps_normal.mp4 \
  -vf 'scale_vaapi=w=1280:h=720' \
  -c:v h264_vaapi \
  /tmp/example.mp4
```

### UniFi Video: transcoding into Apple HomeKit via HomeBridge

| | fps | CPU% | fps/CPU core |
|-:|-|-|-|
| Software: | 15 | 300% | 5 |
| Hardware: | 30<sup>†</sup> | 20% | 150 |
| *Improvement:* | 2x | 15x | 30x |

> <sup>†</sup> Original real-timee stream is 30fps and transcoding is done
in real time. It can not go any faster than 30fps.

Software transcoding example command:
 ```sh
sudo docker run --rm \
  --device /dev/dri:/dev/dri \
  jrottenberg/ffmpeg:vaapi \
  -rtsp_transport http -re -i $UNIFI_VIDEO_CAMERA_RSTP_URL?apiKey=$UNIFI_VIDEO_USER_API_KEY \
  -threads 0 -vcodec libx264 -an -pix_fmt yuv420p -r 30 -f rawvideo -tune zerolatency \
  -vf 'scale=1920:1080' \
  /tmp/example.mp4
```

Hardware-accelerated transcoding example command:
```sh
sudo docker run --rm \
  --device /dev/dri:/dev/dri \
  jrottenberg/ffmpeg:vaapi \
  -hwaccel vaapi -vaapi_device /dev/dri/renderD128 -hwaccel_output_format vaapi \
  -rtsp_transport http -re -i $UNIFI_VIDEO_CAMERA_RSTP_URL?apiKey=$UNIFI_VIDEO_USER_API_KEY \
  -threads 0 -vcodec libx264 -an -pix_fmt yuv420p -r 30 -f rawvideo -tune zerolatency \
  -vf 'scale_vaapi=w=1920:h=1080' \
  -c:v h264_vaapi \
  /tmp/example.mp4
```
