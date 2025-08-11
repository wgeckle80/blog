---
title: Firecracker Guest Debugging With GDB
date: 2025-08-11
categories: [Virtualization, Firecracker]
tags: [firecracker, microvm, gdb, alpine linux, linux, freebsd]
---

In my Google Summer of Code project
[Port FreeBSD to QEMU microvm](https://wiki.freebsd.org/SummerOfCode2025Projects/PortFreeBSDToQEMUMicrovm),
I wanted to analyze the behavior of FreeBSD's Firecracker port in GDB, as QEMU
microvm is a microVM platform in the same vein. The documentation at
<https://github.com/firecracker-microvm/firecracker/blob/main/docs/gdb-debugging.md>
describing how to achieve this was insufficient on its own, as it currently lacks
critical details in the process of getting GDB debugging in Firecracker to work.
This post outlines the process I took to debug FreeBSD guests in Firecracker.


## Setup

To build and run Firecracker, I'm running Alpine Linux in an x86_64 virtual
machine. In particular,

```terminal
$ uname -a
Linux alpine 6.12.40-0-virt #1-Alpine SMP PREEMPT_DYNAMIC 2025-07-24 12:43:02 x86_64 Linux
$ cat /etc/alpine-release
3.22.1
```

Any reasonably modern Linux distribution should work, but I chose Alpine Linux
due to it being lightweight and relatively easy to setup.

Enable the community repository by editing uncommenting

```conf
#http://mirrors.edge.kernel.org/alpine/v3.22/community
```

in `/etc/apk/repositories`. More information can be found at
<https://wiki.alpinelinux.org/wiki/Repositories#Enabling_the_community_repository>.
The required packages for building and running Firecracker, as well as running
GDB, are

```
bash
docker
gdb
git
iproute2
jq
```

Some highly recommended additional packages include

```
ripgrep
rsync
tmux
```

If you are not `root`, reference
<https://wiki.alpinelinux.org/wiki/Setting_up_a_new_user> for user management.
If you created a user during setup, they are automatically part of the `wheel`
group.

Reference <https://wiki.alpinelinux.org/wiki/Docker> to setup Docker as root.
You must add yourself to the `docker` group if you are not `root`, as
Firecracker cannot be built rootless.

Note that, if you're trying to follow the getting started guide at
<https://github.com/firecracker-microvm/firecracker/blob/main/docs/getting-started.md>
to verify that a well-tested configuration works on your Firecracker
installation, you must make modifications to any command involving `grep -oP`
if you're on Alpine Linux, as the `P` option is not available in the BusyBox
version of `grep`. You will also need additional packages I have not listed.

<!-- I have opted to build the Linux kernel and Debian rootfs in a Docker container,
so there is no additional setup in that regard. -->

To build FreeBSD, at least in my experience, it is ideal to have FreeBSD
installed on bare metal or in a virtual machine. I have not tried building
it on macOS, but building it in Debian 12 was broken when I was first learning
FreeBSD back in April for my GSoC project proposal. On the FreeBSD host,
install

```
git
```

Some recommended packages include

```
rsync
tmux
```


## Building Firecracker

Clone the Firecracker source with

```terminal
git clone https://github.com/firecracker-microvm/firecracker.git
```

In the source code's root directory, run

```terminal
rg "cargo build"
```

One of the results should be something like

```terminal
tools/release.sh
136:cargo build --target "$CARGO_TARGET" $CARGO_OPTS --workspace --bins --examples
```

Append `--features "gdb"` to that line.

```
cargo build --target "$CARGO_TARGET" $CARGO_OPTS --workspace --bins --examples --features "gdb"
```

At this point, in the source code's root directory, simply run

```terminal
tools/devtool build
```

As stated in the project's `README.md`, the binary will be placed at
`build/cargo_target/$(uname -m)-unknown-linux-musl/debug/firecracker`. The
`build/cargo_target/$(uname -m)-unknown-linux-musl/debug` directory can
be moved freely, so it's recommended to move it to a location like `~/opt`
and rename it to `firecracker`.


<!-- ## Building and Debugging a Linux Kernel

For this example, I am just using the latest commit of the `master` branch of
the Linux kernel rather than checking out any particular version. The article
[A Guide to Compiling the Linux Kernel All By Yourself](https://itsfoss.com/compile-linux-kernel/)
by [Pratham Patel](https://itsfoss.com/author/pratham/) is particularly helpful
to follow along with in terms of generally building the kernel.

Clone the latest commit of the `master` branch of the Linux kernel, and change
directory into the root of the source.

```terminal
git clone https://github.com/torvalds/linux.git -b master --single-branch --depth 1
cd linux
```

Download the debug configuration from the recommended guest configurations
in the Firecracker repository, and rename it to `.config`.

```terminal
wget https://raw.githubusercontent.com/firecracker-microvm/firecracker/refs/heads/main/resources/guest_configs/debug.config
mv debug.config .config
```


## Building and Debugging a Linux Rootfs -->


## Building and Debugging FreeBSD

For this example, I am just building the latest commit of the `main` branch of
FreeBSD. You can checkout a particular version if needed.

Clone the latest commit of the `main` branch of FreeBSD with

```terminal
git clone https://cgit.freebsd.org/src/ -b main --single-branch --depth 1
```

Ensure you have write access to `/usr/obj`. If you just want to compile the
kernel, run

```terminal
make -j <cpu-cores> buildkernel KERNCONF=FIRECRACKER \
	CONFIG_FRAME_POINTER=y \
	CONFIG_DEBUG_INFO=y \
	CONFIG_SCHED_MC=n \
	CONFIG_SCHED_MC_PRIO=n
```

I'm not sure if the latter four options are required, but since the Firecracker
GDB guide suggested them, I put them in to be safe. After the kernel is built,
you will find `kernel` and `kernel.debug` in
`/usr/obj/<path-to-cloned-repository>/amd64.amd64/sys/FIRECRACKER`. Copy them
to anywhere on the Linux host running Firecracker. Additionally, copy the
FreeBSD source to the exact location it was on FreeBSD during its kernel
build. If the source code was located somewhere in a user's home directory, you
should have a user with the same username on the Linux host. Afterwards, in
the directory containing `kernel` and `kernel.debug`, create a configuration
file.

```json
{
  "boot-source": {
    "kernel_image_path": "kernel",
    "boot_args": "console=ttyS0 reboot=k panic=1 pci=off"
  },
  "drives": [],
  "machine-config": {
    "vcpu_count": 1,
    "mem_size_mib": 512,
    "gdb_socket_path": "/tmp/gdb.socket"
  },
  "logger": {
    "log_path": "firecracker.log",
    "level": "Debug",
    "show_level": true,
    "show_log_origin": true
  },
  "network-interfaces": [
    {
      "iface_id": "net1",
      "guest_mac": "06:00:AC:10:00:02",
      "host_dev_name": "tap0"
    }
  ]
}
```
{: file='config.json' }

For the network interface, ensure the `tun` module is loaded.

```terminal
lsmod | grep tun
```

If the command gives no output, load the module with

```terminal
doas modprobe tun
```

After confirming that the module is loaded, run the script

```bash
#!/bin/bash

TAP_DEV="tap0"
TAP_IP="172.16.0.1"
MASK_SHORT="/30"

# Setup network interface
doas ip link del "$TAP_DEV" 2> /dev/null || true
doas ip tuntap add dev "$TAP_DEV" mode tap
doas ip addr add "${TAP_IP}${MASK_SHORT}" dev "$TAP_DEV"
doas ip link set dev "$TAP_DEV" up

# Enable ip forwarding
doas sh -c "echo 1 > /proc/sys/net/ipv4/ip_forward"
doas iptables -P FORWARD ACCEPT

# This tries to determine the name of the host network interface to forward
# VM's outbound network traffic through. If outbound traffic doesn't work,
# double check this returns the correct interface!
HOST_IFACE=$(ip -j route list default |jq -r '.[0].dev')

# Set up microVM internet access
doas iptables -t nat -D POSTROUTING -o "$HOST_IFACE" -j MASQUERADE || true
doas iptables -t nat -A POSTROUTING -o "$HOST_IFACE" -j MASQUERADE
```
{: file='start_network_interface.sh' }

Assuming the Firecracker binary is located at `~/opt/firecracker/firecracker`,
start Firecracker with

```terminal
doas ~/opt/firecracker/firecracker --no-api --config-file config.json
```

The Firecracker instance should start listening for a GDB connection. Start GDB
in a separate terminal.

```terminal
doas gdb
```

Add the FreeBSD kernel as a symbol file, and connect to the GDB socket.

```terminal
(gdb) add-symbol-file kernel
add symbol table from file "kernel"
(y or n) y
warning: A handler for the OS ABI "FreeBSD" is not built into this configuration
of GDB.  Attempting to continue with the default i386:x86-64 settings.

Reading symbols from kernel...
Reading symbols from <directory-containing-kernel>/kernel.debug...
(gdb) target remote /tmp/gdb.socket
Remote debugging using /tmp/gdb.socket
warning: No executable has been specified and target does not support
determining executable automatically.  Try using the "file" command.
0x0000000000921000 in ?? ()
```

Exiting GDB will end the Firecracker process. If you need to restart
Firecracker, be sure to delete the GDB socket file.

```terminal
doas rm /tmp/gdb.socket
```

Since the kernel of the Firecracker port of FreeBSD is currently broken, I
could not test the use of a FreeBSD rootfs. Everything from here is essentially
speculation based on work done for the FreeBSD QEMU microvm port, and I cannot
currently confirm that the steps I provide for Firecracker actually work.

If you want to add a drive without spending hours compiling a FreeBSD rootfs,
download a VM snapshot UFS raw image. For example, to download a latest FreeBSD
15.0 image, go to
<https://download.freebsd.org/snapshots/VM-IMAGES/15.0-CURRENT/amd64/Latest/>.
After uncompressing the archive, `config.json` should look like

```json
{
  "boot-source": {
    "kernel_image_path": "kernel",
    "boot_args": "console=ttyS0 reboot=k panic=1 pci=off"
  },
  "drives": [
    {
      "drive_id": "rootfs",
      "path_on_host": "FreeBSD-15.0-CURRENT-amd64-ufs.raw",
      "is_root_device": true,
      "is_read_only": false
    }
  ],
  "machine-config": {
    "vcpu_count": 1,
    "mem_size_mib": 512,
    "gdb_socket_path": "/tmp/gdb.socket"
  },
  "logger": {
    "log_path": "firecracker.log",
    "level": "Debug",
    "show_level": true,
    "show_log_origin": true
  },
  "network-interfaces": [
    {
      "iface_id": "net1",
      "guest_mac": "06:00:AC:10:00:02",
      "host_dev_name": "tap0"
    }
  ]
}
```
{: file='config.json' }

Start Firecracker and GDB in the same manner as without a rootfs.

If you want to build a rootfs, it will take significantly longer than just
building the kernel (about 2 hours longer in my case). To do so, on the
FreeBSD host, run

```terminal
make -j <cpu-cores> buildworld buildkernel KERNCONF=FIRECRACKER \
	CONFIG_FRAME_POINTER=y \
	CONFIG_DEBUG_INFO=y \
	CONFIG_SCHED_MC=n \
	CONFIG_SCHED_MC_PRIO=n
cd release
doas make -j <cpu-cores> firecracker DESTDIR=<destination-directory>
```

You will find `freebsd-kern.bin` and `freebsd-rootfs.bin` in
`<destination-direcotry>`. Copy `freebsd-rootfs.bin` to the Linux host.
Modify `config.json` to reflect the change in rootfs.

```json
{
  "boot-source": {
    "kernel_image_path": "kernel",
    "boot_args": "console=ttyS0 reboot=k panic=1 pci=off"
  },
  "drives": [
    {
      "drive_id": "rootfs",
      "path_on_host": "freebsd-rootfs.bin",
      "is_root_device": true,
      "is_read_only": false
    }
  ],
  "machine-config": {
    "vcpu_count": 1,
    "mem_size_mib": 512,
    "gdb_socket_path": "/tmp/gdb.socket"
  },
  "logger": {
    "log_path": "firecracker.log",
    "level": "Debug",
    "show_level": true,
    "show_log_origin": true
  },
  "network-interfaces": [
    {
      "iface_id": "net1",
      "guest_mac": "06:00:AC:10:00:02",
      "host_dev_name": "tap0"
    }
  ]
}
```
{: file='config.json' }

Start Firecracker and GDB in the same manner as without a rootfs.
