---
title: NetBSD microvm and GDB
date: 2025-06-28
categories: [Blog]
tags: [blog]
---

Tom suggested to reference [NetBSD](https://www.netbsd.org/) for an
implementation of proper TSC initialization on QEMU microvm. Before diving into
the source code, I wanted to get NetBSD running in a microvm instance myself.


## smolBSD

[smolBSD](https://www.smolbsd.org/) is a project which greatly simplifies the
process of getting a NetBSD microvm up and running on a Linux or NetBSD host.
Although any reasonable Linux distribution should work with appropriate
packages installed, I chose to spin up a NetBSD virtual machine running on 4
CPU cores, 4096 MB of RAM, and a 20 GB disk image.

Installing NetBSD 10.1 from the ISO image is fairly straightforward. See 
<https://www.netbsd.org/docs/guide/en/chap-exinst.html> for more information on
the installation process. In my particular installation, 1024 MB of disk space
was allocated for swap, and the rest was given to the root directory. I opted
for a full installation, but I believe the only requirement is that the
`X11 sets` are installed, as they're needed to install QEMU. At the end of the
installer, I selected the `Enable installation of binary packages` option to
enable DHCP and install `pkgin` up front. If this is not done up front, see
<https://www.netbsd.org/docs/guide/en/chap-boot.html#chap-boot-pkgsrc> to
install it after finishing the OS install. I also enabled `sshd` and created a
non-root user in the `wheel` group before exiting the installation.

As root, install the required packages by running

```terminal
pkgin install git qemu gmake bsdtar rsync
```

Following the "Fetch a directly bootable kernel" and "Create your own root
filesystem" subsections on the project page, run 

```terminal
git clone https://github.com/NetBSDfr/smolBSD.git
cd smolBSD
gmake rescue
curl -O -L https://smolbsd.org/assets/netbsd-SMOL
shasum -a 256 netbsd-SMOL
```

These commands result in `rescue-amd64.img` and `netbsd-SMOL` appearing in the
cloned repository's root directory. Ensure the output of the last command
matches that on the project page. Also, as root, start `nvmm` with

```terminal
modload nvmm
```

The module is only needed for the smolBSD startup script, so I didn't opt
to load it on startup. After this, per the project page, run

```terminal
./startnb.sh -k netbsd-SMOL -i rescue-amd64.img
```

If everything is successful, we have proof of NetBSD running on QEMU microvm.
I couldn't find a way to exit out of the VM properly, as `shutdown -p now`
triggers the system to reboot. Hence, I prefer to run this command in a `tmux`
session, and kill the window after I'm done messing with the VM.


## NetBSD Kernel Compilation

For our purposes, simply being able to run smolBSD isn't enough. I needed to
build my own kernel with debug symbols for proper debugging. To do so, begin
by cloning the NetBSD source code. If only the latest commit of the `trunk`
branch is needed, run

```terminal
git clone -b trunk --single-branch --depth 1 https://github.com/NetBSD/src.git netbsd-src
```

to reduce the download size and rename the cloned repository directory.
Assuming we want the output files to be placed in `~/obj`, and we want to run
the compilation workload with 4 CPU cores, run

```terminal
./build.sh -U -u -j4 -m amd64 -O ~/obj tools
./build.sh -U -u -j4 -m amd64 -O ~/obj kernel.gdb=MICROVM
```

in the source root directory to build the kernel with debug symbols. See
<https://www.netbsd.org/docs/guide/en/chap-build.html> and `BUILDING` for more
information. Test the built kernel by running

```terminal
./startnb.sh -k ~/obj/sys/arch/amd64/compile/MICROVM/netbsd.gdb -i rescue-amd64.img
```

in the smolBSD root directory created earlier. Additionally, run

```terminal
gdb -q ~/obj/sys/arch/amd64/compile/MICROVM/netbsd.gdb
```

and verify that `l main` prints out some C code to ensure that debug symbols
work properly.


## QEMU Instantiation

In addition to a debuggable kernel, I also needed more fine control of the VM
startup command, as smolBSD's startup script is not sufficient for debugging.
Instead of attempting to reverse engineer smolBSD's startup script,
`sys/arch/amd64/conf/MICROVM` provides a good starting point.

```sh
qemu-system-x86_64                                                    \
      -M microvm,x-option-roms=off,rtc=on,acpi=off,pic=off,accel=kvm  \
      -m 256 -cpu host -kernel ${KERNEL}                              \
      -append "root=ld0a console=com rw -z"                           \
      -display none -device virtio-blk-device,drive=hd0               \
      -drive file=${IMG},format=raw,id=hd0                            \
      -device virtio-net-device,netdev=net0                           \
      -netdev user,id=net0,ipv6=off,hostfwd=::2200-:22                \
      -global virtio-mmio.force-legacy=false -serial stdio
```
{: .nolineno }

I took inspiration from the previous FreeBSD microvm script, in writing a new
one, so the kernel and disk image are provided as two respective command line
arguments. However, I wanted the ability to take out modules by simply
commenting them out, which is not possible without breaking a shell script. I
also wanted an excuse to learn the basics of another programming language, so
I adapted this command into a Perl script called `microvm_debug.pl`.

```perl
#!/usr/pkg/bin/perl

my ($kernel, $img) = @ARGV;

if (not defined $kernel or not defined $img) {
    die "Need kernel and disk image command line arguments";
}

my $memory = "256M";
my $cores = 1;

exec("qemu-system-x86_64",
     "-M", "microvm,x-option-roms=off,rtc=on,acpi=off,pic=off",
     "-m", "$memory",
     "-smp", "$cores",
     "-kernel", "$kernel",
     "-append", "root=ld0a console=com rw -z",
     "-display", "none",
     "-device", "virtio-blk-device,drive=hd0",
     "-drive", "file=$img,format=raw,id=hd0",
     "-device", "virtio-net-device,netdev=net0",
     "-netdev", "user,id=net0,ipv6=off,hostfwd=::2200-:22",
     "-global", "virtio-mmio.force-legacy=false",
     "-serial", "stdio",
     "-s", "-S");
```
{: file='microvm_debug.pl' }

In adapting the script, the virtual accelerator needed to be removed, and the
GDB flags needed to be added. Notably, NetBSD's native hypervisor, NVMM, cannot
be used when debugging an OS in QEMU to my knowledge. This shouldn't be a big
deal, as we also ran FreeBSD in microvm without an accelerator.

Give execution privileges to the script and run

```terminal
./microvm_debug.pl ~/obj/sys/arch/amd64/compile/MICROVM/netbsd.gdb rescue-amd64.img
```

To debug the running kernel instance, run

```terminal
gdb -q ~/obj/sys/arch/amd64/compile/MICROVM/netbsd.gdb -ex "target remote localhost:1234"
```


## GDB Python Build

To my knowledge, every package containing GDB in the NetBSD repositories has
the program built without Python scripting support. While it isn't strictly
necessary, it's useful to automate repetitious actions, and as a
self-documenting way to return to points of interest in the code. Given this,
I sought to compile the program myself.

I started by downloading the source of version 16.3, the current latest
release. This release did not build on NetBSD, and even after a quick
patch, it would not compile with Python support. I reported the
[compilation failure without Python bug](https://sourceware.org/bugzilla/show_bug.cgi?id=33114)
without realizing that a fix has already been implemented in the `master`
branch. Perhaps coincidentally, the current `master` branch also fixed my
ability to compile with Python, though I'm not sure why. The build steps laid
out below are for GDB's `master` branch downloaded on June 27th, 2025, and may
or may not contain a few superfluous steps due to my initial effort to
compile version 16.3.

As root, install a few more required packages.

```terminal
pkgin install gmp mpfr python312 ncurses
```

Create symlinks as root so Python and ncurses can be properly used during
compilation.

```terminal
ln -s /usr/pkg/bin/python3.12 /usr/bin/python
ln -s /usr/pkg/bin/python3.12 /usr/bin/python3
ln -s /usr/pkg/lib/python3.12 /usr/lib
ln -s /usr/pkg/lib/libpython3.12.so /usr/lib
ln -s /usr/pkg/lib/libpython3.12.so.1.0 /usr/lib
ln -s /usr/pkg/include/python3.12 /usr/include
ln -s /usr/pkg/lib/libncurses++.a /usr/lib
ln -s /usr/pkg/lib/libncurses++.la /usr/lib
ln -s /usr/pkg/lib/libncurses++.so /usr/lib
ln -s /usr/pkg/lib/libncurses++.so.6 /usr/lib
ln -s /usr/pkg/lib/libncurses++.so.6.5.0 /usr/lib
ln -s /usr/pkg/lib/libncurses.a /usr/lib
ln -s /usr/pkg/lib/libncurses.la /usr/lib
ln -s /usr/pkg/lib/libncurses.so /usr/lib
ln -s /usr/pkg/lib/libncurses.so.6 /usr/lib
ln -s /usr/pkg/lib/libncurses.so.6.5.0 /usr/lib
ln -s /usr/pkg/include/ncurses /usr/include
```

Configure, build, and install. Installation must be done as root.

```terminal
./configure --with-gmp=/usr/pkg --with-mpfr=/usr/pkg --with-python
gmake -j 4
gmake install
```

Ensure that Python support is working by running `/usr/local/bin/gdb`, running
the GDB command `python-interactive 2 + 2`, and verifying that the result is
`4`. Use an alias to have `gdb` point to this executable.
