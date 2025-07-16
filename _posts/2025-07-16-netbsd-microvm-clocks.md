---
title: NetBSD microvm Clocks
date: 2025-07-16
categories: [Port FreeBSD to QEMU microvm]
tags: [netbsd, microvm, pvclock, tsc, lapic]
---

We explore the NetBSD source code to discover analysis starting points,
investigate the use of TSC in the `MICROVM` build of NetBSD, and begin looking
at LAPIC timer initialization.


## NetBSD Source Exploration

The first thing we want to do is check if PV clock is used at all in the
`MICROVM` build of NetBSD. We start with a recursive grep of `pvclock`.

```terminal
$ rg "pvclock"
sys/external/mit/xen-include-public/dist/xen/include/public/features.h
88:/* x86: pvclock algorithm is safe to use on HVM */
89:#define XENFEAT_hvm_safe_pvclock           9

sys/arch/xen/xen/hypervisor.c
640:    XEN_TST_F(hvm_safe_pvclock);
```

Unfortunately, this isn't too helpful for us. If we try searching for `pv`
instead, we get far too much output to reasonably parse through. We can
try searching in Cscope, but this also yields too many options to comb
through.

We can also try searching for `tsc`. Searching with `rg` gives too much output
to manually analyze, but searching for it in Cscope and auditing the
results gives us the potentially useful files `sys/arch/x86/x86/tsc.c` and
`sys/arch/xen/xen/xen_clock.c`. These, especially the former, could be useful
to take note of.

We can also check `sys/arch/x86/conf/MICROVM.common` to see if there's anything
useful in the kernel configuration. There are a few lines related to Xen.

```conf
# Xen PV support for PVH and HVM guests, needed for PVH boot
options         XENPVHVM
options         XEN
hypervisor*     at mainbus?             # Xen hypervisor
xenbus*         at hypervisor?          # Xen virtual bus
xencons*        at hypervisor?          # Xen virtual console
```
{: file='sys/arch/x86/conf/MICROVM.common' .nolineno }

PCI and ACPI are not used in the build.

```conf
#pci*   at mainbus? bus ?
#acpi0  at mainbus0
```
{: file='sys/arch/x86/conf/MICROVM.common' .nolineno }

There are a few lines related to the PV bus.

```conf
# Virtual bus for non-PCI devices
pv* at pvbus?

## Virtio devices
# Use MMIO by default
virtio* at pv?
```
{: file='sys/arch/x86/conf/MICROVM.common' .nolineno }

Running `rg pvbus` gives `sys/arch/amd64/conf/files.amd64` as a potentially
useful file among several seemingly useless ones for this project.

One more avenue we could take is looking at the initialization sequence.
Looking inside `sys/kern/init_main.c` and searching for `timer` brings us to

```c
	/* Now timer is working.  Enable preemption. */
	kpreempt_enable();
```
{: file='sys/kern/init_main.c' .nolineno }

This is a decent start. We can reduce our search to `time`, and while there
are a few more results to comb through, we find

```c
	/* Initialize timekeeping. */
	time_init();
...
	inittimecounter();
...
	/*
	 * Now that we've found all the hardware, start the real time
	 * and statistics clocks.
	 */
	initclocks();
```
{: file='sys/kern/init_main.c' .nolineno }

This exploration of the NetBSD source code gives us some solid things to keep
in mind.


## TSC Investigation

As a starting point, we turn our attention to `sys/arch/x86/x86/tsc.c`. In
particular, we focus on the function `tsc_tc_init`. Using our previous NetBSD
debugging setup, we set a breakpoint to the function, continue, and look
at the backtrace.

```terminal
(gdb) b tsc_tc_init
Breakpoint 1 at 0xffffffff802586e2: file <netbsd-source>/sys/arch/x86/x86/tsc.c, line 409.
(gdb) c
Continuing.

Breakpoint 1, tsc_tc_init () at <netbsd-source>/sys/arch/x86/x86/tsc.c:409
(gdb) bt 10
#0  tsc_tc_init () at <netbsd-source>/sys/arch/x86/x86/tsc.c:409
#1  0xffffffff802459e6 in cpu_boot_secondary_processors () at <netbsd-source>/sys/arch/x86/x86/cpu.c:788
#2  0xffffffff8042168d in configure2 () at <netbsd-source>/sys/kern/init_main.c:834
#3  main () at <netbsd-source>/sys/kern/init_main.c:574
```

Looking at end of the definition of `cpu_boot_secondary_processors`, we see

```c
	/* Now that we know about the TSC, attach the timecounter. */
	tsc_tc_init();
```
{: file='sys/arch/x86/x86/cpu.c' .nolineno }

I'm not sure why we would know about the TSC at this point. However, it
doesn't seem like it matters, as if we look at the definition of `configure2`,
we see

```c
#if defined(MULTIPROCESSOR)
	cpu_boot_secondary_processors();
#endif
```
{: file='sys/kern/init_main.c' .nolineno }

It's peculiar that `tsc_tc_init` is only being called for multiprocessor
configurations, and that it's buried in a backtrace with `configure2`. There
must be some other timecounter being initialized. If we look at the end of
the definition of `tsc_tc_init`, we see

```c
	if (tsc_freq != 0) {
		tsc_timecounter.tc_frequency = tsc_freq;
		tc_init(&tsc_timecounter);
	}
```
{: file='sys/arch/x86/x86/tsc.c' .nolineno }

We note the generic function `tc_init`, which is defined in
`sys/kern/kern_tc.c`. We can restart the kernel and break on this function to
likely get useful information.

```terminal
(gdb) b tc_init
Breakpoint 1 at 0xffffffff80362871: file <netbsd-source>/sys/kern/kern_tc.c, line 707.
(gdb) c
Continuing.

Breakpoint 1, tc_init (tc=0xffffffff8080a980 <i8254_timecounter>) at <netbsd-source>/sys/kern/kern_tc.c:707
(gdb) bt 10
#0  tc_init (tc=0xffffffff8080a980 <i8254_timecounter>) at <netbsd-source>/sys/kern/kern_tc.c:707
#1  0xffffffff8025fd35 in startrtclock () at <netbsd-source>/sys/arch/x86/isa/clock.c:348
#2  0xffffffff80231892 in cpu_configure () at <netbsd-source>/sys/arch/amd64/amd64/autoconf.c:99
#3  0xffffffff804215fc in configure () at <netbsd-source>/sys/kern/init_main.c:796
#4  main () at <netbsd-source>/sys/kern/init_main.c:551
```

For our purposes, the RTC shouldn't matter, so we move on.

```terminal
(gdb) c
Continuing.

Breakpoint 1, tc_init (tc=0xffffffff8080a640 <lapic_timecounter>) at <netbsd-source>/sys/kern/kern_tc.c:707
(gdb) bt 10
#0  tc_init (tc=0xffffffff8080a640 <lapic_timecounter>) at <netbsd-source>/sys/kern/kern_tc.c:707
#1  0xffffffff8025c9e7 in lapic_initclock () at <netbsd-source>/sys/arch/x86/x86/lapic.c:620
#2  0xffffffff80330e5f in initclocks () at <netbsd-source>/sys/kern/kern_clock.c:245
#3  0xffffffff80421625 in configure2 () at <netbsd-source>/sys/kern/init_main.c:813
#4  main () at <netbsd-source>/sys/kern/init_main.c:574
```

This is intriguing, as our error message in FreeBSD was `Statistical lapic
calibration failed!  Clocks might be ticking at variable rates.` It's still
strange that `configure2` is in the backtrace, but `lapic_initclock` may be of
interest. Before investigating any particular path any further, let's keep
going with hitting the `tc_init` breakpoint until it stops being called.

```terminal
(gdb) c
Continuing.

Breakpoint 1, tc_init (tc=0xffffffff8080fe40 <intr_timecounter>) at <netbsd-source>/sys/kern/kern_tc.c:707
(gdb) bt 10
#0  tc_init (tc=0xffffffff8080fe40 <intr_timecounter>) at <netbsd-source>/sys/kern/kern_tc.c:707
#1  0xffffffff80330ead in initclocks () at <netbsd-source>/sys/kern/kern_clock.c:256
#2  0xffffffff80421625 in configure2 () at <netbsd-source>/sys/kern/init_main.c:813
#3  main () at <netbsd-source>/sys/kern/init_main.c:574
(gdb) c
Continuing.

Breakpoint 1, tc_init (tc=0xffffffff8080a560 <tsc_timecounter>) at <netbsd-source>/sys/kern/kern_tc.c:707
(gdb) bt 10
#0  tc_init (tc=0xffffffff8080a560 <tsc_timecounter>) at <netbsd-source>/sys/kern/kern_tc.c:707
#1  0xffffffff80258783 in tsc_tc_init () at <netbsd-source>/sys/arch/x86/x86/tsc.c:252
#2  0xffffffff802459e6 in cpu_boot_secondary_processors () at <netbsd-source>/sys/arch/x86/x86/cpu.c:788
#3  0xffffffff8042168d in configure2 () at <netbsd-source>/sys/kern/init_main.c:834
#4  main () at <netbsd-source>/sys/kern/init_main.c:574
(gdb) c
Continuing.
^C
Program received signal SIGINT, Interrupt.
0xffffffff8023193f in bus_space_read_stream_1 ()
```

Given the information we already have, these last two backtraces don't seem too
useful. Since we found `lapic_initclock` previously, let's explore LAPIC clock
initialization.


## LAPIC Clock Initialization

Looking at `lapic_initclock`, we find that only the primary CPU can calibrate
and initialize the timer, which makes sense. We also see

```c
		/*
		 * Recalibrate the timer using the cycle counter, now that
		 * the cycle counter itself has been recalibrated.
		 *
		 * Not needed when lapic_per_second is read from CPUID.
		 */
		if (!lapic_from_cpuid)
			lapic_calibrate_timer(true);
```
{: file='sys/arch/x86/x86/lapic.c' .nolineno }

Searching for `lapic_from_cpuid` in Cscope tells us that the variable is only
written to after initialization to `false` in `tsc_freq_vmware_cpuid`, which is
a function that should not be relevant to us. We can confirm that the timer
is recalibrated with a GDB Python script.

```py
import gdb

GDB_PORT = 1234
KERNEL_FILE = "~/obj/sys/arch/amd64/compile/MICROVM/netbsd.gdb"

gdb.execute(f"file {KERNEL_FILE}")
gdb.execute(f"target remote localhost:{GDB_PORT}")

gdb.execute("break lapic_initclock")
gdb.execute("continue")
if not bool(gdb.parse_and_eval("lapic_from_cpuid")):
    print("LAPIC timer calibrated")
else:
    print("LAPIC timer not calibrated")
```
{: file='lapic_does_timer_calibrate.py' }

Sourcing this script confirms that the LAPIC timer is calibrated, so we move
on.

Also in `lapic_initclock`, we see

```c
		/*
		 * Hook up time counter.  This assume that all LAPICs have
		 * the same frequency.
		 */
		lapic_timecounter.tc_frequency = lapic_per_second;
		tc_init(&lapic_timecounter);
```
{: file='sys/arch/x86/x86/lapic.c' .nolineno }

Searching for `lapic_per_second` in Cscope tells us that it's a global variable
initialized to 0 in `sys/arch/x86/x86/lapic.c`. It's overwritten in both
`tsc_freq_cpuid` and `lapic_calibrate_timer`. By setting a breakpoint on the
former and continuing in GDB, we find that the function is never called, so we
can at least isolate this variable's changes to `lapic_calibrate_timer`.
`lapic_per_second` is simply used to set the timecounter frequency before
initializing the LAPIC clock, so we're done getting context for the global
variables used in `lapic_initclock`.

This process funnels us into analyzing `lapic_calibrate_timer`. Looking at the
definition, I don't know what's going on here. The comment above the
function definition might help in some way.

```c
/*
 * Calibrate the local apic count-down timer (which is running at
 * bus-clock speed) vs. the i8254 counter/timer (which is running at
 * a fixed rate).
 *
 * The Intel MP spec says: "An MP operating system may use the IRQ8
 * real-time clock as a reference to determine the actual APIC timer clock
 * speed."
 *
 * We're actually using the IRQ0 timer.  Hmm.
 */
```
{: file='sys/arch/x86/x86/lapic.c' .nolineno }

During my discussion with Tom about this, we found the
[APIC OSDev wiki page](https://wiki.osdev.org/APIC), which directs us to
Chapter 10 of the
[Intel System Programming Guide, Vol 3A Part 1](https://www.intel.com/content/dam/www/public/us/en/documents/manuals/64-ia-32-architectures-software-developer-vol-3a-part-1-manual.pdf).
I'll use this as a starting point to figure out what `lapic_calibrate_timer`
is doing.
