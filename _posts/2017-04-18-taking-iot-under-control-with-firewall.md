---
title: "Taking your LAN network under control in the presence of IoT devices"
---

Using Internet of Things devices like Wi-Fi-connected light bulbs, switches,
motion detectors and such poses a challenge to secure your LAN. Devices'
firmware usually is of a pretty bad quality with administrator password
hard-coded and exploits never patched. At the same time these devices heavily
rely on zero-configuration protocols for ease of discovery, and these protocols
were never developed with security in mind.

I'm trying to reconcile these inherently insecure devices working with
inherently insecure protocols by using a specially configured Linux-based
firewall. I'll guide you through my thought process to show how to make
everything work seamlessly, but (more) secure at the same time.

```txt
               Internet
                  |
(ETH00) ====== Router (192.168.0.1) ====== (192.168.0.X)
  |               |                         Secured Wi-Fi/Eth Network
Firewall          |
| | |          (172.16.0.0)
| | |          Guest Wi-Fi Network
| | (ETH01)
| | (192.168.0.X)
| | DeviceType0
| |
| (WLAN01)
| (192.168.0.X)
| DeviceType1
|
(WLAN00)
(192.168.0.X)
DeviceType2
```


<!--more-->

## Overview

 - All trusted or _Secured_ devices are directly connected to the main _Router_
   connected to the _Internet_ and are not affected by the setup.
   (Example: owner's laptops and phones that have complete implicit trust are
   on the _Secured_ network)
 - All completely untrusted or _Guest_ devices devices live on a separate
   subnetwork and don't pass any traffic to the _Secured_ network.
   This is how Apple AirPort Guest Network works out of the box.
   (Example: A friend's laptop is on the _Guest_ network does not see any other
   devices)
 - All partially trusted or _DeviceTypeX_ devices are connected through the
   _Firewall_ and can only respond to requests from devices from Secured
   network.
   _Secured_ network devices see _DeviceTypeX_ devices, but not in reverse.
   Devices of the same Type can interact with each other.
   (Example: iPhone on the _Secured_ network can see Belkin WeMo switch on
   _DeviceType1_
   newtork, but not in reverse; or Belkin WeMo switch can talk to detector
   on the same _DeviceType1_ network, but can not talk to the HP Printer on
   _DeviceType2_ network).


### Firewall Network Interfaces

Firewall has these network interfaces:
 - ETH00: wired, connects to Router to the Secured network
 - ETH01, ETH02, ETH03: wired, each connect to a specific DeviceType network
 - WLAN00, WLAN01, WLAN02, WLAN03: 4 different Wi-Fi networks sharing the same
   channel and the same network card
 - BRIDGE00: Bridge across _all_ network interfaces. This is the interface
   all `FORWARD` rules would be applied against


## Hardware Used

### PC: [Soekris net6501](http://soekris.com/products/net6501-1.html)

 - Intel Atom CPU 600MHz-1.6GHz
 - 512MB-2048MB RAM
 - 4x Intel 82574L Gigabit Ethernet ports
 - RS-232 Serial port for BIOS and OS boot control
 - PCIe slot for extensions
 - Size: 11.4"W x 6.11"D x 1.41"H
 - Passive cooling

US-made, small box, passive cooling, good networking and relatively cheap.

If you're using macOS, I'd recommend to buy any FTDI RS-232 USB cable
and [Serial App](https://itunes.apple.com/us/app/serial/id877615577?mt=12)
to simplify configuration over a serial port.


### Wi-Fi: [TP-Link N900](http://www.tp-link.com/us/products/details/cat-5519_TL-WDN4800.html)

 - Supports Wi-Fi 802.11n, 2.4GHz and 5GHz
 - 3x3 antenna array
 - Access point: 4 different SSID on the same channel

Really good [Linux support](https://wikidevi.com/wiki/TP-LINK_TL-WDN4800),
relatively fast, cheap.


## Software Used

### [ZeroShell](http://www.zeroshell.org)

 - Small, simple Linux-based firewall/router
 - Web- and SSH-based interfaces, no X11
 - Firewall: full iptables support
 - Wi-Fi: access point w/ many different SSID on the same channel

Simple, ideal for network appliance like Soekris.

Configuration instructions:
 - Create all Wi-Fi networks from _RS-232 \| Wi-Fi manager_, you can have
   up to 4 on the same channel
 - Create _Setup \| Network \| New BRIDGE_ with all interfaces: `BRIDGE00`


#### IPv6

ZeroShell allows all IPv6 traffic by default and there is no configuration.
Simplest way to deal with it is to drop it all.

Add a new script: _Setup \| Scripts/Cron \| NAT And Virtual Servers_

```sh
# Disable IPv6 support: routing and all, ZeroShell does not do it right by default
echo -n "[Disable IPv6]: "
ip6tables -P FORWARD DROP \
&& sysctl -w net.ipv6.conf.all.disable_ipv6=1 \
&& sysctl -w net.ipv6.conf.default.disable_ipv6=1 \
&& sysctl -w net.ipv6.conf.lo.disable_ipv6=1 \
&& echo SUCCESS || echo FAILED
```

> Another distribution I evaluated was pfSense. pfSense 2.3 is built on top of
> FreeBSD 10 and does not support my Wi-Fi card of choice: TL-WDN4800.
> pfSense 2.4 built on top of FreeBSD 11 is in beta test as of April 2017
> and only supports amd64, which requires kernel recompilation to work on
> [Soekris net6501](http://wiki.soekris.info/Installing_FreeBSD#net6501).
>
> pfSense 2.3 installation instructions:
>  - Text on the console would be garbled, press `3` to get into a command line
>    interface of FreeBSD loader
>  - Type
>    [`set console=comconsole`](https://www.freebsd.org/doc/en/books/handbook/serialconsole-setup.html)
>    to fix console. You should see `OK`
>  - Type
>    [`set hint.acpi.0.disabled=1`](https://puck.nether.net/~majdi/ntp/)
>    to disable ACPI. Otherwise boot will hang without any log messages
>  - Type `boot -v` to boot the loader and install as usual


## ZeroShell Firewall Rules

All these rules are configured using ZeroShell Firewall.
_Firewall_ strings represent how the rule _looks_ like in ZeroShell Web UI.
I also provide
relevant _iptables_ rules to make sure you can re-implement them in any
other Linux distribution.

All rules apply to _FORWARD_ chain only.


#### 1. Drop invalid packets

> Technically speaking this rule is not required, but why not?

 - Connection State: _INVALID_
 - ACTION: _DROP_

Firewall: `DROP all opt -- in * out * 0.0.0.0/0 -> 0.0.0.0/0 PHYSDEV match --physdev-is-bridged state INVALID /* Drop invalid packets */`

iptables: `-A FORWARD -m physdev --physdev-is-bridged -m state --state INVALID -j DROP`


### Broadcast Protocols

#### 2. Allow DHCP requests to _Secure_ network

> There is no good way to filter out DHCP requests as they are broadcasts

 - Output: _ETH00_
 - Protocol Matching: _UDP_
 - Source Port: `68`
 - Dest. Port: `67`
 - IPTABLES: `-m pkttype --pkt-type broadcast`
 - ACTION: _ACCEPT_
 - LOG

Firewall: `ACCEPT udp opt -- in * out * 0.0.0.0/0 -> 0.0.0.0/0 PHYSDEV match --physdev-out ETH00 --physdev-is-bridged udp spt:68 dpt:67 PKTTYPE = broadcast /* Allow DHCP requests to Secure network */`

iptables: `-A FORWARD -p udp -m physdev --physdev-out ETH00 --physdev-is-bridged -m udp --sport 68 --dport 67 -m pkttype --pkt-type broadcast -m limit --limit 10/min --limit-burst 15 -j LOG --log-prefix "FORWARD/002"`


#### 3. Drop the rest of the Broadcast

> I'm not sure how ARP is handled, but it looks like it just works

 - IPTABLES: `-m pkttype --pkt-type broadcast`
 - ACTION: _DROP_

Firewall: `DROP 4 opt -- in * out * 0.0.0.0/0 -> 0.0.0.0/0 PHYSDEV match --physdev-is-bridged PKTTYPE = broadcast /* Drop the rest of broadcast (IP w/o ARP) */`

iptables: `-A FORWARD -p ip -m physdev --physdev-is-bridged -m pkttype --pkt-type broadcast -j DROP`


### Multicast ZeroConf Protocols

ZeroConf protocols rely on Multicast IP messages to find devices
available nearby. Two specific protocols I support are
[mDNS](https://en.wikipedia.org/wiki/Multicast_DNS) and
[SSDP](https://en.wikipedia.org/wiki/Simple_Service_Discovery_Protocol).

These protocols are very different from a normal TCP/UDP Unicast as _iptables_
does not know if that's _ESTABLISHED_ or _NEW_ connection. And we only allow
_requests_ from _Secured_ network to _DeviceTypeX_ networks, and only
_responses_ from _DeviceTypeX_ to _Secured_ network.


#### 4. Allow mDNS requests from _Secured_ network

> mDNS search request may contain several requests in the same packet. Any device
> may potentially respond if it knows something:
> - UDP Multicast to `224.0.0.251:5353`
> - Binary UDP data
> - UDP data 17th bit is 0 (3rd byte highest bit)

 - Output: _ETH00_
 - Fragments: Not match second and further
 - IPTABLES: `-m u32 --u32 0>>22&0x3C@8>>15&1=0`
 - nDPI: mdns
 - ACTION: _ACCEPT_

Firewall: `ACCEPT all opt !f in * out * 0.0.0.0/0 -> 0.0.0.0/0 PHYSDEV match --physdev-in ETH00 --physdev-is-bridged ndpi protocol mdns u32 "0x0>>0x16&0x3c@0x8>>0xf&0x1=0x0" /* Allow mDNS requests from Secured network (UDP payload, 3rd byte, first bit is 0) */ `

iptables: `-A FORWARD ! -f -m physdev --physdev-in ETH00 --physdev-is-bridged -m ndpi  --mdns  -m u32 --u32 "0x0>>0x16&0x3c@0x8>>0xf&0x1=0x0" -m limit --limit 10/min --limit-burst 15 -j LOG --log-prefix "FORWARD/004"`


#### 5. Allow mDNS responses to _Secured_ network

> mDNS search response may contain several responses in the same packet.
> Any device may potentially respond if it knows something:
> - UDP Multicast to `224.0.0.251:5353`
> - Binary UDP data
> - UDP data 17th bit is 0 (3rd byte highest bit)

 - Output: _ETH00_
 - Fragments: Not match second and further
 - IPTABLES: `-m u32 --u32 0>>22&0x3C@8>>15&1=1`
 - nDPI: mdns
 - ACTION: _ACCEPT_
 - LOG

Firewall: `ACCEPT all opt !f in * out * 0.0.0.0/0 -> 0.0.0.0/0 PHYSDEV match --physdev-out ETH00 --physdev-is-bridged ndpi protocol mdns u32 "0x0>>0x16&0x3c@0x8>>0xf&0x1=0x1" /* Allow mDNS responses to Secured network (UDP payload, 3rd byte, first bit is 1) */`

iptables: `-A FORWARD ! -f -m physdev --physdev-in ETH00 --physdev-is-bridged -m ndpi  --mdns  -m u32 --u32 "0x0>>0x16&0x3c@0x8>>0xf&0x1=0x0" -j ACCEPT`


#### 6. Allow SSDP requests from _Secured_ network

> SSDP search request uses
> [HTTPU](https://en.wikipedia.org/wiki/Universal_Plug_and_Play#Protocol)
> as a transport layer:
>  - UDP Multicast to `239.255.255.250:1900`
>  - From `A.B.C.D:N`
>  - Text UDP data
>  - Starts with `M-SEARCH *`

 - Output: _ETH00_
 - Fragments: Not match second and further
 - IPTABLES: `-m string --algo bm --to 0x40 --string M-SEARCH`
 - nDPI: ssdp
 - ACTION: _ACCEPT_
 - LOG

Firewall: `ACCEPT all opt !f in * out * 0.0.0.0/0 -> 0.0.0.0/0 PHYSDEV match --physdev-out ETH00 --physdev-is-bridged ndpi protocol mdns u32 "0x0>>0x16&0x3c@0x8>>0xf&0x1=0x1" /* Allow mDNS responses to Secured network (UDP payload, 3rd byte, first bit is 1) */`

iptables: `-A FORWARD ! -f -m physdev --physdev-in ETH00 --physdev-is-bridged -m ndpi  --ssdp  -m string --string "M-SEARCH" --algo bm --to 64 -m limit --limit 10/min --limit-burst 15 -j LOG --log-prefix "FORWARD/006"`


#### 7. Allow SSDP responses to _Secured_ network

> SSDP search response uses
> [HTTPU](https://en.wikipedia.org/wiki/Universal_Plug_and_Play#Protocol)
> as a transport layer:
>  - UDP Unicast to `A.B.C.D:N`
>  - Text UDP data
>  - Starts with `HTTP/1.1 200 OK`

> SSDP responses uses _Unicast_ and not _Multicast_. And destination
> port depends on the original SSDP request port number

> We ignore SSDP notifications (essentially broadcast messages):
>  - UDP Multicast to `239.255.255.250:1900`
>  - Text UDP data
>  - Starts with `NOTIFY *`

 - Output: _ETH00_
 - Fragments: Not match second and further
 - Protocol Matching: _UDP_
 - IPTABLES: `-m string --algo bm --to 0x40 --string HTTP/1.1`
 - nDPI: ssdp
 - ACTION: _ACCEPT_
 - LOG

Firewall: `ACCEPT udp opt !f in * out * 0.0.0.0/0 -> 0.0.0.0/0 PHYSDEV match --physdev-out ETH00 --physdev-is-bridged STRING match "HTTP/1.1" ALGO name bm TO 64 /* Allow SSDP responses to Secured network (UDP unicast) */`

iptables: `-A FORWARD -p udp ! -f -m physdev --physdev-out ETH00 --physdev-is-bridged -m string --string "HTTP/1.1" --algo bm --to 64 -m limit --limit 10/min --limit-burst 15 -j LOG --log-prefix "FORWARD/007"`


#### 8. Drop the rest of the Multicast

 - IPTABLES: `-m pkttype --pkt-type multicast`
 - ACTION: _DROP_

Firewall: `DROP all opt -- in * out * 0.0.0.0/0 -> 0.0.0.0/0 PHYSDEV match --physdev-is-bridged PKTTYPE = multicast /* Drop the rest of multicast */`

iptables: `-A FORWARD -m physdev --physdev-is-bridged -m pkttype --pkt-type multicast -j DROP`


### Unicast Network Protocols

There are many Unicast protocols that we should take into account. Rules of
thumb:
 - Connections originating from _Secured_ network are allowed
 - Responses to a connection from _Secured_ network are allowed
 - Connections to _Router_ are usually allowed
 - Connections to _Internet_ are always allowed


#### 9. Allow all traffic from _Secured_ network

 - Input: ETH00
 - ACTION: _ACCEPT_

Firewall: `ACCEPT all opt -- in * out * 0.0.0.0/0 -> 0.0.0.0/0 PHYSDEV match --physdev-in ETH00 --physdev-is-bridged /* Allow all traffic from Secured network */`

iptables: `-A FORWARD -m physdev --physdev-in ETH00 --physdev-is-bridged -j ACCEPT`


#### 10. Allow all established and related packets

 - Output: _ETH00_
 - Connection state: ESTABLISHED, RELATED
 - ACITON: ACCEPT

Firewall: `ACCEPT all opt -- in * out * 0.0.0.0/0 -> 0.0.0.0/0 PHYSDEV match --physdev-out ETH00 --physdev-is-bridged state RELATED,ESTABLISHED /* Allow all established and related packets */`

iptables: `-A FORWARD -m physdev --physdev-out ETH00 --physdev-is-bridged -m state --state RELATED,ESTABLISHED -j ACCEPT`


#### 11. Allow an ICMP ping traffic to _Router_

> Some devices rely on ping to the gateway to verify a connection status

> Router address is hardcoded here to 192.168.0.1

 - Output: _ETH00_
 - Destination IP: 192.168.0.1
 - Protocol Matching: ICMP
 - ICMP Type: echo-request (ping)
 - ACTION: _ACCEPT_
 - LOG

Firewall: `ACCEPT icmp opt -- in * out * 0.0.0.0/0 -> 192.168.0.1 PHYSDEV match --physdev-out ETH00 --physdev-is-bridged icmptype 8 /* Allow an ICMP ping traffic to AirPort */`

iptables: `-A FORWARD -d 192.168.0.1/32 -p icmp -m physdev --physdev-out ETH00 --physdev-is-bridged -m icmp --icmp-type 8 -m limit --limit 10/min --limit-burst 15 -j LOG --log-prefix "FORWARD/011"`


#### 12. Allow any UDP traffic to _Router_ (DHCP, DNS etc.)

> Router address is hardcoded here to 192.168.0.1

 - Output: _ETH00_
 - Destination IP: 192.168.0.1
 - Protocol Matching: _UDP_
 - ACTION: _ACCEPT_
 - LOG

Firewall: `ACCEPT udp opt -- in * out * 0.0.0.0/0 -> 192.168.0.1 PHYSDEV match --physdev-out ETH00 --physdev-is-bridged /* Allow any UDP traffic to AirPort (DHCP, DNS etc.) */`

iptables: `-A FORWARD -d 192.168.0.1/32 -p udp -m physdev --physdev-out ETH00 --physdev-is-bridged -m limit --limit 10/min --limit-burst 15 -j LOG --log-prefix "FORWARD/012"`


#### 13. Allow any traffic outside of LAN network

> We rely on our LAN to be limited to 192.168.0.0/16 network

 - Output: _ETH00_
 - Destination IP: 192.168.0.0/16
 - ACTION: _ACCEPT_

Firewall: `ACCEPT all opt -- in * out * 0.0.0.0/0 !-> 192.168.0.0/16 PHYSDEV match --physdev-out ETH00 --physdev-is-bridged /* Allow any traffic outside of LAN network */`

iptables: `-A FORWARD ! -d 192.168.0.0/16 -m physdev --physdev-out ETH00 --physdev-is-bridged -j ACCEPT`


#### 14. DROP by default

 - Policy: DROP

iptables: `-P FORWARD DROP`



## End result: _iptables_ rules

Here is an end result in form of _iptables_ rules which you can use directly
if desired.

`iptables -S`

```sh
-P INPUT ACCEPT
-P FORWARD DROP
-P OUTPUT ACCEPT
-N NetBalancer
-N SYS_DNS
-N SYS_GUI
-N SYS_HTTPS
-N SYS_INPUT
-N SYS_OUTPUT
-N SYS_SSH
-A INPUT -j SYS_GUI
-A INPUT -j SYS_INPUT
-A INPUT -p tcp -m tcp --dport 80 -j SYS_HTTPS
-A INPUT -p tcp -m tcp --dport 443 -j SYS_HTTPS
-A INPUT -p tcp -m tcp --dport 22 -j SYS_SSH
-A FORWARD -m physdev --physdev-is-bridged -m state --state INVALID -j DROP
-A FORWARD -p udp -m physdev --physdev-out ETH00 --physdev-is-bridged -m udp --sport 68 --dport 67 -m pkttype --pkt-type broadcast -m limit --limit 10/min --limit-burst 15 -j LOG --log-prefix "FORWARD/002"
-A FORWARD -p udp -m physdev --physdev-out ETH00 --physdev-is-bridged -m udp --sport 68 --dport 67 -m pkttype --pkt-type broadcast -j ACCEPT
-A FORWARD -p ip -m physdev --physdev-is-bridged -m pkttype --pkt-type broadcast -j DROP
-A FORWARD ! -f -m physdev --physdev-in ETH00 --physdev-is-bridged -m ndpi  --mdns  -m u32 --u32 "0x0>>0x16&0x3c@0x8>>0xf&0x1=0x0" -m limit --limit 10/min --limit-burst 15 -j LOG --log-prefix "FORWARD/004"
-A FORWARD ! -f -m physdev --physdev-in ETH00 --physdev-is-bridged -m ndpi  --mdns  -m u32 --u32 "0x0>>0x16&0x3c@0x8>>0xf&0x1=0x0" -j ACCEPT
-A FORWARD ! -f -m physdev --physdev-out ETH00 --physdev-is-bridged -m ndpi  --mdns  -m u32 --u32 "0x0>>0x16&0x3c@0x8>>0xf&0x1=0x1" -m limit --limit 10/min --limit-burst 15 -j LOG --log-prefix "FORWARD/005"
-A FORWARD ! -f -m physdev --physdev-out ETH00 --physdev-is-bridged -m ndpi  --mdns  -m u32 --u32 "0x0>>0x16&0x3c@0x8>>0xf&0x1=0x1" -j ACCEPT
-A FORWARD ! -f -m physdev --physdev-in ETH00 --physdev-is-bridged -m ndpi  --ssdp  -m string --string "M-SEARCH" --algo bm --to 64 -m limit --limit 10/min --limit-burst 15 -j LOG --log-prefix "FORWARD/006"
-A FORWARD ! -f -m physdev --physdev-in ETH00 --physdev-is-bridged -m ndpi  --ssdp  -m string --string "M-SEARCH" --algo bm --to 64 -j ACCEPT
-A FORWARD -p udp ! -f -m physdev --physdev-out ETH00 --physdev-is-bridged -m string --string "HTTP/1.1" --algo bm --to 64 -m limit --limit 10/min --limit-burst 15 -j LOG --log-prefix "FORWARD/007"
-A FORWARD -p udp ! -f -m physdev --physdev-out ETH00 --physdev-is-bridged -m string --string "HTTP/1.1" --algo bm --to 64 -j ACCEPT
-A FORWARD -m physdev --physdev-is-bridged -m pkttype --pkt-type multicast -j DROP
-A FORWARD -m physdev --physdev-in ETH00 --physdev-is-bridged -j ACCEPT
-A FORWARD -m physdev --physdev-out ETH00 --physdev-is-bridged -m state --state RELATED,ESTABLISHED -j ACCEPT
-A FORWARD -d 192.168.0.1/32 -p icmp -m physdev --physdev-out ETH00 --physdev-is-bridged -m icmp --icmp-type 8 -m limit --limit 10/min --limit-burst 15 -j LOG --log-prefix "FORWARD/011"
-A FORWARD -d 192.168.0.1/32 -p icmp -m physdev --physdev-out ETH00 --physdev-is-bridged -m icmp --icmp-type 8 -j ACCEPT
-A FORWARD -d 192.168.0.1/32 -p udp -m physdev --physdev-out ETH00 --physdev-is-bridged -m limit --limit 10/min --limit-burst 15 -j LOG --log-prefix "FORWARD/012"
-A FORWARD -d 192.168.0.1/32 -p udp -m physdev --physdev-out ETH00 --physdev-is-bridged -j ACCEPT
-A FORWARD ! -d 192.168.0.0/16 -m physdev --physdev-out ETH00 --physdev-is-bridged -j ACCEPT
-A OUTPUT -j SYS_OUTPUT
-A SYS_DNS -s 10.0.0.0/8 -j ACCEPT
-A SYS_DNS -s 172.16.0.0/12 -j ACCEPT
-A SYS_DNS -s 192.168.0.0/16 -j ACCEPT
-A SYS_DNS -s 192.168.0.0/24 -j ACCEPT
-A SYS_DNS -s 192.168.0.0/24 -j ACCEPT
-A SYS_DNS -s 192.168.250.0/24 -j ACCEPT
-A SYS_DNS -j DROP
-A SYS_GUI -s 192.168.0.2/32 -p tcp -m tcp --dport 12081 -j ACCEPT
-A SYS_HTTPS -i lo -j ACCEPT
-A SYS_HTTPS -m physdev --physdev-in ETH00 -j ACCEPT
-A SYS_HTTPS -j DROP
-A SYS_INPUT -i lo -j ACCEPT
-A SYS_INPUT -p udp -m udp --sport 53 -m state --state ESTABLISHED -j ACCEPT
-A SYS_INPUT -p tcp -m tcp --sport 53 -m state --state ESTABLISHED -j ACCEPT
-A SYS_INPUT -p udp -m udp --dport 53 -j SYS_DNS
-A SYS_INPUT -p tcp -m tcp --dport 53 -j SYS_DNS
-A SYS_INPUT -p tcp -m tcp --sport 80 -m state --state ESTABLISHED -j ACCEPT
-A SYS_INPUT -p tcp -m tcp --sport 8245 -m state --state ESTABLISHED -j ACCEPT
-A SYS_INPUT -p udp -m udp --sport 123 -m state --state ESTABLISHED -j ACCEPT
-A SYS_INPUT -j RETURN
-A SYS_OUTPUT -o lo -j ACCEPT
-A SYS_OUTPUT -p udp -m udp --dport 53 -j ACCEPT
-A SYS_OUTPUT -p tcp -m tcp --dport 80 -j ACCEPT
-A SYS_OUTPUT -p tcp -m tcp --dport 8245 -j ACCEPT
-A SYS_OUTPUT -p udp -m udp --dport 123 -j ACCEPT
-A SYS_OUTPUT -j RETURN
-A SYS_SSH -i lo -j ACCEPT
-A SYS_SSH -m physdev --physdev-in ETH00 -j ACCEPT
-A SYS_SSH -j DROP
```

`ip6tables -S`

```sh
-P INPUT ACCEPT
-P FORWARD DROP
-P OUTPUT ACCEPT
```
