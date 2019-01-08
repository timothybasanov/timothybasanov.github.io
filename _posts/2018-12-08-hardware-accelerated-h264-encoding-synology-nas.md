---
title:  "Hardware-Accelerated h264 Encoding on Synology NAS"
---

> Updated after publishing: I've got reports of verified support on DS218+ and DS418play. 
I've added Debian Stretch-specific instructions. Added a disclaimer. Opened a pull request
[#30 for homebridge-camera-ffmpeg-ufv](https://github.com/gozoinks/homebridge-camera-ffmpeg-ufv/pull/30)
to add support for VAAPI-based video transcoding.

> Disclaimer: I know very little about _ffmpeg_ and video encoding. I played
around for several days to figure out how to make hardware video transcoding
to work and just wrote down my findings. I'd be happy if somebody who knows
knows about these things would help me to better understand why things
behave the way they do.

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
also has QuickSync, all of this applies to DS218+
([verified by ArtisanalCollabo](https://www.reddit.com/r/synology/comments/a4jboo/you_can_use_hardwareaccelerated_h264_encoding_on/ebg9hsg/))
and DS418play (verified by Arsen Vartapetov).

<!--more-->

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

### Docker-based _ffmpeg_ and _VAAPI_

> Check [_VAAPI_ documentation](https://trac.ffmpeg.org/wiki/Hardware/VAAPI)
for all the internal details, I would only show a very short summary.

_VAAPI_ is a magical API that allows _ffmpeg_ to use hardware acceleration
for different video-related operations and works across different hardware.
It's always one of `/dev/dri/*` devices that can be used to talk to
the underlying hardware. We only need one for our purposes:
`/dev/dri/renderD128` (literally, `D128` is the same across platforms).

Options to add to enable _VAAPI_:
  * Enable _VAAPI_ `-hwaccel vaapi`
  * Make frame buffer format conversion to make hardware codec happy:
    `-hwaccel_output_format vaapi` or
    `-vf 'format=nv12,hwupload'` or `-vf 'scale_vaapi=w=1280:h=720'`
  * Actually use h264-codec with _VAAPI_: `-c:v h264_vaapi`

Simplest command to verify your encoding performance
using an example video [Big Buck Bunny](http://bbb3d.renderfarming.net):
```sh
sudo docker run --rm \
  --device /dev/dri:/dev/dri \
  jrottenberg/ffmpeg:vaapi \
  -hwaccel vaapi -hwaccel_output_format vaapi \
  -i http://distribution.bbb3d.renderfarming.net/video/mp4/bbb_sunflower_1080p_30fps_normal.mp4 \
  -c:v h264_vaapi \
  /tmp/example.mp4
```

> Check a longer example in a performance section below.

### Debian Stretch-based _ffmpeg_ issues

Debian Stretch-based _ffmpeg_ 3.2 differs from Docker defaults and
require additional tweaks to make it work.
I've only verified it from within Debian-based
Docker image with _ffmpeg_ installed via `apt-get install ffmpeg`, so your
results may differ.

  * VAAPI-based surface format is not supported, so we can not use
    `-hwaccel_output_format vaapi` directly
  * This means we need to download decoded frames into memory and upload
    them back via `-vf 'format=nv12,hwupload'`
  * We need to explicitly specify device to upload frames to via
    `-vaapi_device /dev/dri/renderD128`
  * Overall it's much slower than full-speed hardware encoding,
    but it's still much faster than a software one

```sh
sudo docker run --rm \
  --device /dev/dri:/dev/dri \
  debian:stretch-slim \
  /bin/sh -c "
    apt-get update
    apt-get install --assume-yes ffmpeg
    ffmpeg \
      -hwaccel vaapi \
      -vaapi_device /dev/dri/renderD128 \
      -i http://distribution.bbb3d.renderfarming.net/video/mp4/bbb_sunflower_1080p_30fps_normal.mp4 \
      -vf 'format=nv12,hwupload' \
      -c:v h264_vaapi \
      /tmp/example.mp4
  "
```

  * In some cases you can use this format without crashes
    `-hwaccel_output_format vaapi -vf 'format=nv12|vaapi,hwupload'`
    this variant has the same performance as hardware variant,
    but I'm not sure how portable it is

```sh
sudo docker run --rm \
  --device /dev/dri:/dev/dri \
  debian:stretch-slim \
  /bin/sh -c "
    apt-get update
    apt-get install --assume-yes ffmpeg
    ffmpeg \
      -hwaccel vaapi -hwaccel_output_format vaapi \
      -vaapi_device /dev/dri/renderD128 \
      -i http://distribution.bbb3d.renderfarming.net/video/mp4/bbb_sunflower_1080p_30fps_normal.mp4 \
      -vf 'format=nv12|vaapi,hwupload' \
      -c:v h264_vaapi \
      /tmp/example.mp4
  "
```

> You may get this warning: _Hardware accelerated decoding with frame
threading is known to be unstable and its use is discouraged._
The way I read it it should be fixed if I specify `-threads 1`, but it does
not fix it. This transcode was stable enough for my purposes, so I just
ignored it.

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

> Note: If you run your Dockerized app under non-priviledged user, don't
forget to give access to your devices: `chmod 777 /dev/dri/renderD128`

There is no simple way of calling Dockerized _ffmpeg_ from
an another Docker image, but if you use Debian-based docker image chances
are it would be as easy as:
```sh
apt-get install ffmpeg
```

This may pull in the up-to-date version of _ffmpeg_ with all the right bindings
and devices.

## Transcoding Performance Results on DS718+

### Big Buck Bunny: scaling from 1080p into 720p

| | fps | CPU% | fps/CPU core |
|-:|-|-|-|
| Software: | 30 | 380% | 8 |
| Mixed1: | 40 | 70% | 60 |
| Mixed2:<sup>†</sup> | 60 | 70% | 85 |
| Hardware: | 110 | 85% | 130 |
| *Improvement:* | 3x | 5x | 15x |

> <sup>†</sup> For some reason hardware-only surface formats are not supported
on Debian and one needs to copy data between decoder and encoder via a main
memory. This is not Debian-specific, but it only affected my Debian-based
Docker images for some reason. I may be mistaken and it could be that
either encoder or decoder are run in software.

Software transcoding example command:
```sh
sudo docker run --rm \
  --device /dev/dri:/dev/dri \
  jrottenberg/ffmpeg:vaapi \
  -i http://distribution.bbb3d.renderfarming.net/video/mp4/bbb_sunflower_1080p_30fps_normal.mp4 \
  -vf 'scale=1280:720' \
  /tmp/example.mp4
```

Mixed1: Transcoding that uses hardware decoder and encoder, but copies data
over through a main memory between them. This is what you get by default
on Debian Stretch. Example command:
```sh
sudo docker run --rm \
  --device /dev/dri:/dev/dri \
  debian:stretch-slim \
  /bin/sh -c "
    apt-get update
    apt-get install --assume-yes ffmpeg
    ffmpeg \
      -hwaccel vaapi \
      -vaapi_device /dev/dri/renderD128 \
      -i http://distribution.bbb3d.renderfarming.net/video/mp4/bbb_sunflower_1080p_30fps_normal.mp4 \
      -vf 'format=nv12,hwupload,scale_vaapi=w=1280:h=720' \
      -c:v h264_vaapi \
      /tmp/example.mp4
  "
```

Mixed2: Just like Mixed1, but does one additional hack with `nv12|vaapi`.
Example command:
```sh
sudo docker run --rm \
  --device /dev/dri:/dev/dri \
  debian:stretch-slim \
  /bin/sh -c "
    apt-get update
    apt-get install --assume-yes ffmpeg
    ffmpeg \
      -hwaccel vaapi -hwaccel_output_format vaapi \
      -vaapi_device /dev/dri/renderD128 \
      -i http://distribution.bbb3d.renderfarming.net/video/mp4/bbb_sunflower_1080p_30fps_normal.mp4 \
      -vf 'format=nv12|vaapi,hwupload,scale_vaapi=w=1280:h=720' \
      -c:v h264_vaapi \
      /tmp/example.mp4
  "
```

Hardware-accelerated transcoding example command:
```sh
sudo docker run --rm \
  --device /dev/dri:/dev/dri \
  jrottenberg/ffmpeg:vaapi \
  -hwaccel vaapi -hwaccel_output_format vaapi \
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
  -hwaccel vaapi -hwaccel_output_format vaapi \
  -rtsp_transport http -re -i $UNIFI_VIDEO_CAMERA_RSTP_URL?apiKey=$UNIFI_VIDEO_USER_API_KEY \
  -threads 0 -vcodec libx264 -an -pix_fmt yuv420p -r 30 -f rawvideo -tune zerolatency \
  -vf 'scale_vaapi=w=1920:h=1080' \
  -c:v h264_vaapi \
  /tmp/example.mp4
```
