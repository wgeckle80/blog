---
title: Initial Setup
date: 2025-06-11
categories: [Blog]
tags: [blog]
---

This post discusses relevant development environment information and initial
progress on this project.


## Development Environment

I'm running [GhostBSD](https://www.ghostbsd.org/) 25.01-R14.2p1 on a
[Lenovo V15 G4 IRU](https://www.lenovo.com/gb/en/p/laptops/lenovo/v-series/lenovo-v15-gen-4-15-inch-intel/83a1002wuk)
laptop. Besides Wi-Fi, most facilities appear to work perfectly out of the box.
To use the RTL8852BE Wi-Fi controller, I installed the
`wifi-firmware-rtw89-kmod-rtw8852b` package and rebooted.

A notable difference from FreeBSD is that GhostBSD doesn't come with utilities
to compile code or ports out of the box, as mentioned in their
[FAQ](https://ghostbsd-documentation-portal.readthedocs.io/en/latest/user/FAQ.html#can-i-use-linux-software-on-my-ghostbsd-system).
They need to be installed with

```terminal
sudo pkg install -g 'GhostBSD*-dev'
```

The installation of other standard FreeBSD packages is described in
<https://docs.freebsd.org/en/books/handbook/ports/>.

The code for this project lives at
<https://github.com/WGeckle80/freebsd-src/tree/microvm-port>. The custom kernel
config `QEMUMICROVM` lives at `sys/amd64/conf/QEMUMICROVM`, which is currently
the same as the `FIRECRACKER` config with a line commented out. It will likely
change over time to suit the port's needs. Since the output files of a kernel
compilation are placed in `/usr/obj`, and it's not always ideal to compile the
kernel with elevated privileges, Tom suggested to change the directory's owner
to my main user with

```terminal
sudo chown <user> /usr/obj
```

After this, a test compilation can be performed with

```terminal
make -j 16 -s buildkernel KERNCONF=QEMUMICROVM TARGET=amd64
```

If successful, the file
`/usr/obj/<path-to-cloned-repository>/amd64.amd64/sys/QEMUMICROVM/kernel`
should exist. The kernel is compiled with debug symbols, so using GDB to
debug a running QEMU instance is easy. When launching the host virtual
machine, append `-s -S` to the other parameters, which initializes a GDB
server on `localhost:1234`. Launch GDB and run

```terminal
target remote localhost:1234
add-symbol-file <path-to-kernel-file>
```

After accepting the prompt, debug as usual.

An initial kernel build takes me about 500 seconds according to the
console output, while subsequent builds finish in a few seconds.


## Initial Progress

This section directly builds off of Tom Jones'
[devlog post](https://adventurist.me/posts/00320), as I've essentially
replicated his findings.

Before any work is done, the `BOOTVERBOSE` macro in `sys/kern/init_main.c` is
set to `1` for potentially useful messages down the line.

The first snag I ran into while attempting to replicate the devlog results
deals with the following code in the `tsc_freq_cpuid_vm` function of
`sys/x86/x86/tsc.c`:

```c
	if (hv_high < 0x40000010)
		return (false);
	do_cpuid(0x40000010, regs);
```

It was mentioned that "Skipping this check get us to the same
`pvclock_get_timecount` fault as when we skipped the assert entirely", but it
wasn't specifically mentioned that the first two lines were commented out.

Following this, I had issues with the Xen configuration description. After
commenting out the `xentimer` line in the kernel configuration file, Tom
mentioned that "fixing up some parts of the Xen `init_ops` lets us advance to
a fascinating panic". He has since provided me with the diff file regarding
`sys/x86/xen/pv.c`, which changed the initialization of `xen_pvh_init_ops`
to

```c
struct init_ops xen_pvh_init_ops = {
	.parse_preload_data		= xen_pvh_parse_preload_data,
#if 0
	.early_clock_source_init	= xen_clock_init,
	.early_delay			= xen_delay,
#endif
	.parse_memmap			= pvh_parse_memmap,
};
```

In `hammer_time_xen`, the `init_ops` initialization was changed to

```c
#if 0
	init_ops = xen_pvh_init_ops;
#else
	init_ops.parse_preload_data = xen_pvh_parse_preload_data,
	init_ops.parse_memmap = pvh_parse_memmap,
#endif
```

Finally, it wasn't clear to me where the file `test.raw` came from. I can at
least get a working VM disk image similar to `test.raw` by downloading
a FreeBSD snapshot. In particular, I downloaded the 15.0-CURRENT UFS raw image
from
<https://download.freebsd.org/snapshots/VM-IMAGES/15.0-CURRENT/amd64/Latest/>.
I should note that I tried the ZFS raw image, but FreeBSD microvm instance
reports a corrupted superblock.

My final launch script looks like

```sh
#!/bin/sh

kernel=$1
disk=$2

memory=512m
cores=4
netif=tap0

bootargs="vfs.root.mountfrom=ufs:/dev/vtbd0p4"

qemu-system-x86_64 -M microvm                      \
    -cpu max                                       \
    -m ${memory}                                   \
    -smp ${cores}                                  \
    -kernel ${kernel}                              \
    -append ${bootargs}                            \
    -nodefaults                                    \
    -no-user-config                                \
    -nographic                                     \
    -serial stdio                                  \
    -drive id=test,file=${disk},format=raw,if=none \
    -device virtio-blk-device,drive=test           \
    -machine acpi=off
```

It is invoked with

```terminal
./microvm.sh <path-to-kernel-file> <path-to-raw-disk-image>
```

My final conclusions are similar to Tom's. Booting up is slow---particularly
during uart0 initialization---and I do get a login screen when using a UFS raw
disk image from the FreeBSD snapshots.
