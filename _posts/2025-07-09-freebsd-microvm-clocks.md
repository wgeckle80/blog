---
title: FreeBSD microvm Clocks
date: 2025-07-09
categories: [Port FreeBSD to QEMU microvm]
tags: [freebsd, microvm, pvclock, tsc, lapic, cpuid]
---

This post discusses the current state of TSC and PV clock usage in the
`MICROVM` build of FreeBSD, relays the basics of the `cpuid` instruction in
x86, and gives a brief overview of TSC and PV clock in x86.


## PV Clock Investigation

We start with the following query in the FreeBSD source root directory:

```terminal
grep -R "pvclock\.c"
```

As of now, besides some binary files we don't care about, `sys/conf/files.x86`
is the only file returned in the query. Looking into the config, we note two
important lines:

```conf
x86/x86/pvclock.c		optional	kvm_clock | xenhvm
x86/x86/tsc.c			standard
```
{: file='sys/conf/files.x86' .nolineno }

In `sys/amd64/conf/MICROVM`, we see that `kvm_clock` is already used in the
build.

```conf
device		kvm_clock
```
{: file='sys/amd64/conf/MICROVM' .nolineno }

Let's reintroduce the `xentimer` device in `sys/amd64/conf/MICROVM`.

```conf
device		xentimer		# Xen x86 PV timer device
```
{: file='sys/amd64/conf/MICROVM' .nolineno }

We can write a Python script to set a breakpoint on every function in
`sys/x86/x86/pvclock.c`.

```py
import gdb

GDB_PORT = 1234
KERNEL_FILE = "/usr/obj/<path-to-freebsd-source>/amd64.amd64/sys/MICROVM/kernel"

PVCLOCK_FUNCTIONS = [
    "pvclock_resume",
    "pvclock_tsc_freq",
    "pvclock_read_time_info",
    "pvclock_read_wall_clock",
    "pvclock_getsystime",
    "pvclock_get_timecount",
    "pvclock_get_wallclock",
    "pvclock_cdev_open",
    "pvclock_cdev_mmap",
    "pvclock_tc_get_timecount",
    "pvclock_tc_vdso_timehands",
    "pvclock_tc_vdso_timehands32",
    "pvclock_gettime",
    "pvclock_init",
    "pvclock_destroy",
]

gdb.execute(f"target remote localhost:{GDB_PORT}")
gdb.execute(f"add-symbol-file {KERNEL_FILE}")

for func in PVCLOCK_FUNCTIONS:
    gdb.execute(f"break {func}")
```
{: file='pvclock_breakpoints.py' }

We also need a script to enable GDB debugging on a FreeBSD microvm guest,
which we can get by slightly modifying our microvm launch script from before.

```sh
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
    -machine acpi=off                              \
    -s -S
```
{: file='microvm_debug.sh' }

We use the Python script by running `microvm_debug.sh` in one window,
opening GDB in another, and executing the command

```terminal
source gdb_script.py
```

Unfortunately, none of the breakpoints are triggered when continuing after
sourcing the script. In fact, for whatever reason, just like Tom was initially,
we are once again stopped at the assertion

```c
	KASSERT((cpu_feature & CPUID_TSC) != 0 && tsc_freq != 0,
	    ("TSC not initialized"));
```
{: file='sys/x86/x86/local_apic.c' .nolineno }

on line 547 of `sys/x86/x86/local_apic.c`. Since `CPUID_TSC` is defined as
`0x00000010`, we can set a breakpoint at the assert in GDB and run

```terminal
p/x cpu_feature & CPUID_TSC
```

The output is `0x10`, so the VM is equipped with TSC. Hence, the issue is
the TSC frequency being set to 0, and we can look in the function
`tsc_freq_cpuid_vm` to investigate.

```c
static int
tsc_freq_cpuid_vm(void)
{
	u_int regs[4];

	if (vm_guest == VM_GUEST_NO)
		return (false);
	//if (hv_high < 0x40000010)
		//return (false);

	do_cpuid(0x40000010, regs);
	tsc_freq = (uint64_t)(regs[0]) * 1000;
	tsc_early_calib_exact = 1;
	return (true);
```
{: file='sys/x86/x86/tsc.c' .nolineno }

Looking at `do_cpuid`, it's simply an assembly instruction.

```c
static __inline void
do_cpuid(u_int ax, u_int *p)
{
	__asm __volatile("cpuid"
	    : "=a" (p[0]), "=b" (p[1]), "=c" (p[2]), "=d" (p[3])
	    :  "0" (ax));
}
```
{: file='sys/amd64/include/cpufunc.h' .nolineno }

We can find information on the `cpuid` instruction at
<https://wiki.osdev.org/CPUID>, <https://www.felixcloutier.com/x86/cpuid>, and
<https://tizee.github.io/x86_ref_book_web/instruction/cpuid.html>. The latter
two pages feature C-like psuedocode for what the function outputs in
`eax`, `ebx`, `ecx`, and `edx` for some `eax` input cases. However, we don't
get any information about an input of `0x40000010`. From the surrounding
context in this particular application, we should be getting some form of the
TSC frequency in `regs[0]` after executing the instruction. Of course, the
register value is 0, and no amount of scaling will make it not zero, so we fail
the check expecting `tsc_freq` to be nonzero.

However, this `tsc_freq` variable doesn't seem to do anything to stop us from
booting except for causing the `KASSERT` to fail. If we set a breakpoint,
set `tsc_freq` to a nonzero value, and continue, the OS proceeds with a slow
boot to the login screen. With this information, we can edit our Python
script from before.

```py
import gdb

GDB_PORT = 1234
KERNEL_FILE = "/usr/obj/<path-to-freebsd-source>/amd64.amd64/sys/MICROVM/kernel"

PVCLOCK_FUNCTIONS = [
    "pvclock_resume",
    "pvclock_tsc_freq",
    "pvclock_read_time_info",
    "pvclock_read_wall_clock",
    "pvclock_getsystime",
    "pvclock_get_timecount",
    "pvclock_get_wallclock",
    "pvclock_cdev_open",
    "pvclock_cdev_mmap",
    "pvclock_tc_get_timecount",
    "pvclock_tc_vdso_timehands",
    "pvclock_tc_vdso_timehands32",
    "pvclock_gettime",
    "pvclock_init",
    "pvclock_destroy",
]

gdb.execute(f"target remote localhost:{GDB_PORT}")
gdb.execute(f"add-symbol-file {KERNEL_FILE}")

for func in PVCLOCK_FUNCTIONS:
    gdb.execute(f"break {func}")

gdb.execute("break local_apic.c:547")
gdb.execute("continue")
gdb.execute("set tsc_freq = 1")
gdb.execute("continue")
```
{: file='pvclock_breakpoints.py' }

None of the `pvclock*` breakpoints are tripped, so this at least tells us that
PV clock isn't utilized in the build. This is true regardless of whether the
`xentimer` device is included in the config file, that change from earlier
has been reverted.

We proceed to the login prompt just as before. However, just after `uart0` is
initialized, the time it takes for
`Statistical lapic calibration failed!  Clocks might be ticking at variable rates.`
to print is much greater than everything else, leading Tom to believe that
resolving this error message is what the rest of this project will be spent on.


## CPUID Vendor ID

Out of curiosity, let's see the information we get from setting the `cpuid`
input to `0x80000000`.

```py
import gdb

GDB_PORT = 1234
KERNEL_FILE = "/usr/obj/<path-to-freebsd-source>/amd64.amd64/sys/MICROVM/kernel"

gdb.execute(f"target remote localhost:{GDB_PORT}")
gdb.execute(f"add-symbol-file {KERNEL_FILE}")

gdb.execute("break tsc_freq_cpuid_vm")
gdb.execute("continue")
for _ in range(3):
    gdb.execute("nexti")
gdb.execute("set $eax = 0x80000000")
gdb.execute("nexti")

print("EAX: " + hex(int(gdb.parse_and_eval("$eax"))))
ebx_bytes = int(gdb.parse_and_eval("$ebx")).to_bytes(4, "little")
ecx_bytes = int(gdb.parse_and_eval("$ecx")).to_bytes(4, "little")
edx_bytes = int(gdb.parse_and_eval("$edx")).to_bytes(4, "little")
print("Vendor ID String: "
      + ebx_bytes.decode()
      + edx_bytes.decode()
      + ecx_bytes.decode())
```
{: file='cpuid_0x80000000.py' }

The last two lines of output are

```terminal
EAX: 0x8000000a
Vendor ID String: AuthenticAMD
```

Unfortunately, this isn't useful information, as the vendor ID string is
already printed during initialization. In fact, soon after
`Origin="AuthenticAMD"` is printed as part of the CPU information,
`Hypervisor: Origin = "TCGTCGTCGTCG"` is printed, which we know denotes the
QEMU vendor ID string from the OSDev wiki page linked earlier.


## x86 Clocks

According to [Wikipedia](https://en.wikipedia.org/wiki/Time_Stamp_Counter), the
TSC is a 64-bit register on x86 processors which keeps track of the number of
CPU cycles since reset.

Even though there is no author and the publish date is March 1, 2014, the blog
post at
<https://cyberdeed.wordpress.com/2014/03/01/virtualization-mode-hvm-pv-pvhvm-pvh/>
is a decent summary of virtualization modes, and I have cross referenced it with
other sources to ensure accuracy. A paravirtualized machine knows its running in a
virtual environment, and is provided hooks to the host system's resources through
specialized interfaces. This technique reduces the need to emulate every physical
device the virtual guest requires, which may have significant performance gains
due to the cost of emulation. PV Clock is a timing mechanism based off this idea.

QEMU provides a list of paravirtualized KVM features
at <https://www.qemu.org/docs/master/system/i386/kvm-pv.html>. On the surface,
it seems we can only take advantage of these in Linux, since KVM is a Linux module.
Of course, I would need to dig deeper to know if this is actually the case.


## Errata

I have changed the config `sys/amd64/conf/QEMUMICROVM` to
`sys/amd64/conf/MICROVM` to align with NetBSD's microvm implementation.
Previous posts referencing this file have been updated accordingly.
