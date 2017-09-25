---
title:  "Installing UniFi and mFI Controller Software onto Ubiquiti's UNC-NVR"
---

It makes some sense to keep all Ubiquiti's software on the same PC,
and UNV-NVR is the best one with software already preinstalled.
NVR has Debian 7 installed by default, so it's easy to install both
mFI and UniFi,
with a caveat of using MongoDB 2.4 instead of a default 2.0 or 3.0 one.

<!--more-->

## Enable SSH on NVR

We will need SSH access to NVR box. You will have an option to disable it
after we finished.

> See Ubituiti's support articles
> [UniFi Video - How to Enable SSH on the UVC-NVR (Hardware NVR)](https://help.ubnt.com/hc/en-us/articles/222970307-UniFi-Video-How-to-Enable-SSH-on-the-UVC-NVR-Hardware-NVR-).

Open http://NVR_IP in Google Chrome, click on "Settings" button up right
and enable SSH server on Configuration page. Don't forget to change the password
from the default ont.

## Install Ubiquiti's software

Connect to the box with a `root` user and the password you setup above.

### Add Ubiquiti's Debian repositories

We need to add references to Ubiquiti's repos to be able to install its
software.

> See Ubiquiti's support articles
> [mFi - How To Install the mFi Controller in Ubuntu or Debian](https://help.ubnt.com/hc/en-us/articles/211656797-mFi-How-To-Install-the-mFi-Controller-in-Ubuntu-or-Debian)
> and
> [UniFi - How to Install & Update via APT on Debian or Ubuntu](https://help.ubnt.com/hc/en-us/articles/220066768-UniFi-How-to-Install-Update-via-APT-on-Debian-or-Ubuntu).
> You never need to update "keys", they come pre-installed.

```sh
echo 'deb http://dl.ubnt.com/mfi/distros/deb/debian debian ubiquiti' \
  > /etc/apt/sources.list.d/ubnt-mfi.list
echo 'deb http://www.ubnt.com/downloads/unifi/debian stable ubiquiti' \
  > /etc/apt/sources.list.d/ubnt-unifi.list
apt-get update
```

### Install MongoDB 2.4

Well, turns out MongoDB 2.0 is too old and 2.4 is required. We need to
install fresher version from
[Wheezy Backports Debian](https://backports.debian.org/Instructions/):

```sh
## Remove MongoDB's old version 2.0 and all Ubiquiti's software
apt-get purge '^mongodb.*' mfi unifi unifi-video
reboot
```
> You need purge to make sure that old MongoDB databases would be disabled.

```sh
## Install a version 2.4
echo 'deb http://ftp.debian.org/debian wheezy-backports main contrib non-free' \
  > /etc/apt/sources.list.d/debian-backports.list
apt-get update
apt-get -t wheezy-backports install mongodb
```

> Using old version of MongoDB almost works, until server reboot.

### Install UniFi, UniFi Video and mFI

Time to reinstall all the software back!

```sh
apt-get install mfi unifi unifi-video
reboot
```

> Be sure to agree for a unifi's backup, if apt asks for permission.
> Declining may cause installation failure and half-installed packages.

That's it! Now everything should be just working. =)

> Note: There is a bug with default permissions for UniFi package. mFI package
is fine. Wrong permissions prevent UniFi being detected by NVR software
and no "button" appears on the main Web UI. This is a temporary fix until a next
restart:
```sh
chmod +x /usr/lib/unifi/data
chmod +r /usr/lib/unifi/data/system.properties
```
