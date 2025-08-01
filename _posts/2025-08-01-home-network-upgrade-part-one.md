---
title: Home Network Upgrade Part One
date: 2025-08-01
categories: [Homelab]
tags: [homelab, opnsense, openwrt, tftp]
---

Currently, our network infrastructure can roughly be laid out as

```
XFINITY 500 mbps download; 20 mbps upload
├── Motorola MB8611 Cable Modem
│   ├── AmpliFi Router HD with 2 associated mesh points
│   │   ├── Printer
│   │   ├── TVs
│   │   ├── Many IoT devices
│   │   ├── UniFi AP AC Pro
│   │   │   ├── Computers
│   │   │   ├── Phones
│   │   │   └── A few IoT devices
│   │   ├── Unmanaged Switch
│   │   │   └── Computers
```

At some point after our router has been running a long time without
being reset, I noticed that our internet was particularly slow, even on wired
connections. Of course, the solution to this was to just restart the router,
but it got me thinking. I just had to guess the particular issues our router
had. Was the CPU being overloaded? Was the router running out of memory?
Was it something else I didn't think of? Additionally, it got me to revisit
the thought of how asinine it is that major router configurations such as
port forwarding and device listings were tied to a mobile phone app, instead
of just being available on the HTTP site served at the gateway. Given my
tenure as a system administrator at [UMIACS](https://www.umiacs.umd.edu/), as
well as my many years of experience in Linux, I figured it's finally time I do
something about this.

The first thing to do is gather information about our current network setup.
Immediately, it caught my eye that our router was severely out of date. It was
on version 3.7.1, while the latest version is currently 4.0.3. After updating,
I ran some tests. For all of the following metrics, I used
<https://www.speedtest.net/>. On a wired connection directly to the router, the
rough metrics I got were

> Download: 90 mbps  
> Upload: 19 mbps

Given that our plan's peak download speed is 500 mbps, it was alarming that
peak wired throughput is less than a fifth of that. After this test, I also
wanted to test wireless performance of just the router and mesh points. By the
router itself, I got

> Download: 80 mbps  
> Upload: 19 mbps

By one of the mesh points at the opposite point in the house, I got

> Download: 93 mbps  
> Upload: 20 mbps

It was surprising that performance was better around the mesh points, so I
unplugged the
[dedicated access point](https://store.ui.com/us/en/products/uap-ac-pro) for a
second and ran another test.

> Download: 94 mbps  
> Upload: 18 mbps

Ideally, due to the nature of the software, it would be ideal if we could
remove this device entirely from the network. However, the figures show that
the mesh points are too useful to consider this, given that they're
software-locked to the router. Add on the fact that we have IoT devices such
as cameras which need to operate on the opposite side of the house, we need
to keep these nodes alive. I would love to install
[OpenWrt](https://openwrt.org/) to improve administration, but the
[guide](https://blog.compardre.com/install-openwrt-amplifi-hd-router-detailed-tutorial/)
I found on this suggests a particularly intrusive method, and there is no
guide on OpenWrt's website to perform the installation. The
[OpenWrt guide on Wi-Fi roaming](https://openwrt.org/docs/guide-user/network/wifi/roaming)
suggests to me that the mesh points are either using 802.11r, 802.11k, or
802.11v, but I don't know these protocols well enough to determine for sure,
given that Ubiquiti doesn't specify the protocol. Regardless, due to the
crippled wired download bandwidth and administrative limitations, it's not
ideal to have the AmpliFi Router HD act as our primary home router, even if
we need to keep it in our network until I learn more about Wi-Fi roaming.
The solution is to set it in bridge mode, which I didn't realize until later.

Given the administrative limitations of the AmpliFi router, I want to keep
our dedicated access point in the network for now. Before, it was just sitting
on a stack of papers in the same room as the router. Performance was okay in
the room itself, but running speed tests in my bedroom on my laptop yielded
peak download speeds below 30 mbps on both the 2.4 and 5 GHz bands. The
solution was to mount the access point on the ceiling, as I'm now getting about
40 mbps on the 5 GHz band. Signal integrity isn't great, but I think that's due
to my laptop's wireless adapter being weak, as my phone picks up wireless
signals from my room just fine. The full-home coverage is generally much worse
than the AmpliFi router, but the ability to separate wireless traffic between
user devices on this access point and IoT devices on the router when I have
a proper router setup makes it worth keeping the access point, even if
it slightly decreases peak wireless bandwidth close to the router.

It was at this point I wanted to test another router. We have a
[Dell Inspiron One 2020](https://www.dell.com/support/product-details/en-us/product/inspiron-one-20-2020-aio/overview)
which has been used sparingly over the past decade, so I figured it would be
the easiest way to quickly try out [OPNsense](https://opnsense.org/). The
builtin ethernet port acted as the WAN port, and a USB to ethernet adapter
acted as the LAN port. The hard drive was previously replaced with an SSD, so I
just installed OPNsense, performed basic configuration, added an additional
unmanaged switch connected to the LAN port since the one currently in our setup
is full, and temporarily replaced our router. Running wired speed tests again,
the results were intriguing.

> Download: 240 mbps  
> Upload: 19 mbps

Even though the USB ports on the device are capped at USB 2.0 speeds, this
setup yielded an increase in peak download bandwidth of over 200% compared to
the [AmpliFi router](https://www.amplifi.com/amplifi-hd). I wasn't even trying
to make a good setup, and yet I got this much of a performance improvement.
This was extremely promising.

The next thing I did was look through some of the old routers we kept but
haven't used in years to see what I could salvage. One of them was a
[Buffalo WZR-600DHP](https://buffaloamericas.com/images/uploads/AirStation_HighPower_N600_DD-WRT_Datasheet.pdf).
This interested me because it ran [DD-WRT](https://dd-wrt.com/) out of the box,
even if the software hasn't been updated since 2016. I first attempted to flash
this with OpenWrt over the web interface.
Unfortunately, this failed, as after the interface told me the firmware
flashed, the device rebooted into DD-WRT. I looked at their
[guide](https://openwrt.org/toh/buffalo/wzr-hp-ag300h) to flashing it over TFTP,
but the guide confused me, as I had never worked with TFTP before. Thankfully,
I found <https://leo.leung.xyz/wiki/WZR-600DHP>, which helped me get OpenWrt
onto the router in a very straightforward manner over TFTP. After some
basic configuration, I dropped this in as our main router. Running a wired
speed test again, I got

> Download: 150 mbps  
> Upload: 19 mbps

I guess this result was predictable, as I believe the router was released in
2012, and the CPU is a single core MIPS processor running at 680 MHz. Still,
it outperformed our current router, so I figured this route could also be
worth pursuing.

The next router I tried was an
[ASUS RT-AC68R](https://dlcdnets.asus.com/pub/ASUS/wireless/RT-AC68R/E8616_RT_AC68R_QSG_English.pdf),
which is the same as the
[RT-AC68U](https://www.asus.com/us/networking-iot-servers/wifi-routers/asus-wifi-routers/rtac68u/).
At least in my experience, the [guide](https://openwrt.org/toh/asus/rt-ac68u)
to flash it with OpenWrt was much easier to follow than the one for the
Buffalo router. After some initial configuration, I dropped this in as our
router, and the wired internet test speeds blew me away.

> Download: 330-340 mbps  
> Upload: 19 mbps

This was over a 300% peak download bandwidth increase over the AmpliFi router.
I was astonished. Given these metrics, I also wanted to test the wireless
access point to see if there were any improvements there too. While the figures
didn't suggest a significant improvement in wireless performance, they were
still better than with the AmpliFi router. This sold me on completing the
router transformation. This is where I realized I could put the AmpliFi router
in bridge mode, so I tested the whole network with that and everything seemed
to work.

I really liked having a Linux machine as our router. I could easily run
`tcpdump` to analyze network traffic. I could get performance metrics on the
HTTP site, and if I wanted to be fancy about it, I could run `btop`. I could
run `tmux` to make command line multitasking a breeze. I didn't have to rely on
a mobile phone app for any kind of management. The experience was liberating.

Unfortunately, problems arose when we turned on the living room TV. Even though
our phones and my laptop could connect to the wireless network of the AmpliFi
router just fine, the TV detected no internet. Since we essentially needed the
TV to work right then and there, this meant I had to revert to using the
AmpliFi router as our main router. I haven't had a chance to diagnose the issue
as of now.

However, this may have been a blessing in disguise, as I realized a few things
after this incident.

1. I didn't see a **Network > Switch** option in the web UI to setup VLANs like
   I did with the Buffalo router. I want VLANs, as I at least want IoT devices
   isolated in their own segment of the network.
2. I want to run a VPN such as WireGuard. OPNsense having that option
   immediately in its web UI is very appealing in that regard, as I haven't
   setup my own VPN before.
3. I could likely use the mini PC I was going to use as an OpenBSD web server
   as an OPNsense router, and there's still more than enough CPU and memory
   to run the server as a [bhyve](https://bhyve.org/) guest. I don't
   anticipate the server getting a large amount of traffic.

The next day, I realized that the wireless access point was probably severely
out of date. I don't remember the particular version it was, but it was on
about version 6.2, when the latest version is currently 6.6.77. However, unlike
the AmpliFi router, this device isn't tied to anything else, so I flashed it
with OpenWrt. The device doesn't serve an HTTP server with the stock firmware,
so it's nice that OpenWrt does to make management more accessible. By default,
OpenWrt makes the device act like a router on `192.168.1.1`, but it's easy to
change this so the router is a DHCP client access point. I mostly followed
[this guide](https://openwrt.org/docs/guide-user/network/wifi/wifiextenders/bridgedap#method_2configuration_via_luci)
to configure the LAN port and
[this guide](https://openwrt.org/docs/guide-quick-start/basic_wifi) to setup
the Wi-Fi network, but I set the protocol to **DHCP client** instead of
**Static address**.

After running speed tests on both the 2.4 and 5 GHz bands, neither showed any
decrease in peak download bandwidth compared to the stock firmware. I could
have analyzed radio wave behavior, but I don't have an SDR, and this
would be more effort than I'm willing to put in right now.

The next steps are currently to figure out why the living room TV is detecting
no internet on the AmpliFi router's wireless network when it's in bridge mode,
and setup the mini PC with OPNsense with ample testing to confirm network
improvements.
