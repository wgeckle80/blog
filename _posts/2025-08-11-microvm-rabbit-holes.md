---
title: MicroVM Rabbit Holes
date: 2025-08-11
categories: [Port FreeBSD to QEMU microvm]
tags: [x86, netbsd, microvm, lapic]
---

This post describes a few of the rabbit holes I've been going down over the
past several weeks regarding this project.


## Local APIC Deep Dive

In the `lapic_calibrate_timer` function of the NetBSD source code, the first
section regarding writing to local APIC registers is

```c
	/*
	 * Configure timer to one-shot, interrupt masked,
	 * large positive number.
	 */
	x86_disable_intr();
	lapic_writereg(LAPIC_LVT_TIMER, LAPIC_LVT_MASKED);
	lapic_writereg(LAPIC_DCR_TIMER, LAPIC_DCRT_DIV1);
	lapic_writereg(LAPIC_ICR_TIMER, 0x80000000);
	(void)lapic_gettick();
```
{: file='sys/arch/x86/x86/lapic.c' .nolineno }

My first thought was to ask if there was a specific reason for `0x80000000` to
be written to the register pointed to by `LAPIC_ICR_TIMER`. We find that
`LAPIC_ICR_TIMER` is defined as `0x380`. Searching for `380H` in the Intel
manual introduced last time leads us to Table 10-1, where we see
that the offset corresponds to the Initial Count Register for the local APIC
timer. This is reinforced in its third occurrence in Table 10-6. Its second
appearance is in Figure 10-11, which tells us that there is no significance
for `0x80000000` being written as the initial count except for it being a
sufficiently large number, which corresponds to the comment above this section.
Nearby is Figure 10-10, which tells us about the Divide Counter Register and
its relation to the local APIC clock. It isn't interesting that we're dividing
the clock by 1 in the above code fragment, so we move on.

Before we move onto `LAPIC_LVT_TIMER`, since the section is short, we can check
out subsection 10.4.2 just below Table 10-1. This section is about verifying
the presence of the local APIC using `cpuid`. We can make a small adaptation to
a FreeBSD debugging script we wrote earlier to verify the presence of the local
APIC in our microvm environment.

```py
import gdb

GDB_PORT = 1234
KERNEL_FILE = "/usr/obj/<path-to-freebsd-source>/amd64.amd64/sys/MICROVM/kernel"

LAPIC_PRESENT_BIT = 9

gdb.execute(f"target remote localhost:{GDB_PORT}")
gdb.execute(f"add-symbol-file {KERNEL_FILE}")

gdb.execute("break tsc_freq_cpuid_vm")
gdb.execute("continue")
for _ in range(3):
    gdb.execute("nexti")
gdb.execute("set $eax = 1")
gdb.execute("nexti")

if int(gdb.parse_and_eval("$edx")) & (1 << LAPIC_PRESENT_BIT):
    print("Local APIC present")
else:
    print("Local APIC not present")
```
{: .file='cpuid_lapic_presence.py' }

As expected, the local APIC is detected by `cpuid`.

Actually, given that we just looked at FreeBSD again, it's important to bring
up something that occurred during a call with Tom. When I was talking about
local APIC register writes initially, he thought I was referring to the FreeBSD
source instead of the NetBSD source. I didn't think anything of the fact that
he searched for `38H` in the Intel manual instead of `380H`. This is an
important distinction, as we saw in Table 10-6 that one column had addresses
for x2APIC mode, and another had addresses for xAPIC mode. If we look into the
function `lapic_write32` in `sys/x86/x86/local_apic.c`, we see that FreeBSD may
be operating in x2APIC mode. If we check section 10.3 of the manual, we see

> The basic operating mode of the xAPIC is xAPIC mode. The x2APIC architecture
> is an extension of the xAPIC architecture, primarily to increase processor
> addressability. The x2APIC architecture provides backward compatibility to
> the xAPIC architecture and forward extendability for future Intel platform
> innovations. These extensions and modifications are supported by a new mode
> of execution (x2APIC mode) are detailed in Section 10.12.

Of course, we need to check which mode(s) are actually being used in our
`MICROVM` kernel. We can write another GDB script to do this.

```py
import gdb

GDB_PORT = 1234
KERNEL_FILE = "/usr/obj/<path-to-freebsd-source>/amd64.amd64/sys/MICROVM/kernel"

gdb.execute(f"target remote localhost:{GDB_PORT}")
gdb.execute(f"add-symbol-file {KERNEL_FILE}")

gdb.execute("break lapic_write32")
gdb.execute("break local_apic.c:547")
gdb.execute("continue")
gdb.execute("set tsc_freq = 1")

while True:
    gdb.execute("continue")
    if bool(gdb.parse_and_eval("x2apic_mode")):
        print("x2APIC mode")
    else:
        print("xAPIC mode")
```
{: .file='x2apic_mode.py' }

This tells us immediately that we're in xAPIC mode, but there are way too many
calls to `lapic_write32` to prove that we're always in xAPIC mode with this
script. To cover all bases, we modify the script to help prove that we never
enter x2APIC during initialization.

```py
import gdb

GDB_PORT = 1234
KERNEL_FILE = "/usr/obj/<path-to-freebsd-source>/amd64.amd64/sys/MICROVM/kernel"

gdb.execute(f"target remote localhost:{GDB_PORT}")
gdb.execute(f"add-symbol-file {KERNEL_FILE}")

gdb.execute("break lapic_write32")
gdb.execute("break local_apic.c:547")
gdb.execute("continue")
gdb.execute("set tsc_freq = 1")

try:
    while True:
        gdb.execute("continue")
        if bool(gdb.parse_and_eval("x2apic_mode")):
            print("One instance of being in x2APIC mode")
            break
except KeyboardInterrupt:
    print("Could not find an instance of being in x2APIC mode")
```
{: .file='x2apic_mode.py' }

It takes awhile to get to the login screen, but eventually we get there
without ever being in x2APIC mode. The NetBSD microvm port is seeming more
useful to us as a result, and now we know we can safely disregard anything
regarding x2APIC in FreeBSD.

This is actually a terrible approach, and we could have just set a watch point.
We'll do that for NetBSD just to confirm our suspicions there.

```terminal
(gdb) watch x2apic_mode
Hardware watchpoint 1: x2apic_mode
(gdb) c
Continuing.
^C
Program received signal SIGINT, Interrupt.
0xffffffff8023210e in x86_stihlt ()
(gdb) p x2apic_mode
$1 = false
```

Going through the datasheet, we see the Local APIC Version Register contents
in Figure 10-7. We can apply the mask `0x00ff00ff` to get the information we
need from it. The only thing then is to actually acquire the value in both
FreeBSD and NetBSD. In both operating systems, we cannot simply break on the
respective local APIC read functions and set their argument `reg`
appropriately, as `reg` is not an lvalue. We can't even set `rax` to a certain
value, as both implementations are optimized around reading from addresses
known at compile time. In FreeBSD, a quick `rg "LAPIC_VERSION"` shows us where
we can set a breakpoint, which we could implement in a script.

```py
import gdb

GDB_PORT = 1234
KERNEL_FILE = "/usr/obj/<path-to-freebsd-source>/amd64.amd64/sys/MICROVM/kernel"

LAPIC_VERSION_MASK = 0x00ff00ff

gdb.execute(f"target remote localhost:{GDB_PORT}")
gdb.execute(f"add-symbol-file {KERNEL_FILE}")

gdb.execute("break local_apic.c:521")
gdb.execute("break local_apic.c:547")
gdb.execute("continue")
gdb.execute("set tsc_freq = 1")
gdb.execute("continue")

print("Lapic version register contents: "
      + hex(int(gdb.parse_and_eval("ver")) & LAPIC_VERSION_MASK))
```
{: file='lapic_version.py' }

Unfortunately, the breakpoint on line 521 of `sys/x86/x86/local_apic.c` doesn't
trip. Additionally, running `rg "LAPIC_VER"` in the NetBSD source shows us that
`LAPIC_VERS` is defined as `0x030`, but it's never referenced in the code.
Given this, we need to make a temporary patch to both operating systems to
print the relevant local APIC version information. In FreeBSD, we add

```c
volatile uint32_t lapic_version = lapic_read32(LAPIC_VERSION) & 0x00ff00ff;
printf("Lapic version: %x\n", lapic_version);
```
{: file='sys/x86/x86/local_apic.c' .nolineno }

somewhere before the `KASSERT` we keep failing is tripped. This tells us that
the register value is `0x50014` in FreeBSD. Meanwhile, in NetBSD, we add

```c
volatile uint32_t lapic_version = lapic_readreg(LAPIC_VERS) & 0x00ff00ff;
printf("Lapic version: %x\n", lapic_version);
```
{: .nolineno }

somewhere reasonable. This tells us that the register value is identical in
NetBSD. It seems like local APIC properties between the two operating systems
are the same.

What's potentially more interesting is the LVT Timer Register. Of course, its
last three digits correspond to the NetBSD definition of `LAPIC_LVT_TIMER` as
`0x320`. The register value is initially set to `1 << 16`, which is equal to
`0x0001 0000`. If we look at Figure 10-8 in the Intel manual, we see that the
value of the register after reset is `0001 0000H`, and bit 16 is the mask bit.
It makes sense that we'd want to mask interrupts here, but this point in the
code turns out to not be too interesting.

The next point in the NetBSD code which has explicit register writes is when a
nonzero local APIC frequency has been found.

```c
	lapic_writereg(LAPIC_LVT_TIMER, LAPIC_LVT_TMM_PERIODIC
	    | LAPIC_LVT_MASKED | LAPIC_TIMER_VECTOR);
	lapic_writereg(LAPIC_DCR_TIMER, LAPIC_DCRT_DIV1);
	lapic_writereg(LAPIC_ICR_TIMER, lapic_tval);
```
{: file='sys/arch/x86/x86/lapic.c' .nolineno }

With context given by the manual, it's easier to note from the macro names that
the local APIC timer is configured to count every second and not trigger
interrupts. I'm currently unsure why the interrupt vector is defined as `0xc0`,
but that particular detail doesn't seem to be important at the moment.


## FreeBSD Statistical Calibration Failure

Something I forgot to do previously was run a backtrace when the message
`Statistical lapic calibration failed! Clocks might be ticking at variable rates.`
came up in FreeBSD. Before running a backtrace, we need to find where this is
printed in the code. Trying to find the full phrase doesn't give anything, but
the output of `rg "Statistical"` includes

```terminal
sys/kern/subr_clockcalib.c
114:                    printf("Statistical %s calibration failed!  "
```

We can write a script to set a breakpoint at this line, as we know it's after
failed assert.

```py
import gdb

GDB_PORT = 1234
KERNEL_FILE = "/usr/obj/<path-to-freebsd-source>/amd64.amd64/sys/MICROVM/kernel"

gdb.execute(f"target remote localhost:{GDB_PORT}")
gdb.execute(f"add-symbol-file {KERNEL_FILE}")

gdb.execute("break subr_clockcalib.c:114")

gdb.execute("break local_apic.c:547")
gdb.execute("continue")
gdb.execute("set tsc_freq = 1")
gdb.execute("continue")
```
{: file='lapic_statistical_calibration_fail.py' }

Executing this script, we get

```terminal
(gdb) source statistical_calibration_fail.py
warning: No executable has been specified and target does not support
determining executable automatically.  Try using the "file" command.
0x000000000000fff0 in ?? ()
add symbol table from file "/usr/obj/<path-to-freebsd-source>/amd64.amd64/sys/MICROVM/kernel"
warning: remote target does not support file transfer, attempting to access files from local filesystem.
Breakpoint 1 at 0xffffffff8095c5e6: file <path-to-freebsd-source>/sys/kern/subr_clockcalib.c, line 114.
Breakpoint 2 at 0xffffffff809545d3: file <path-to-freebsd-source>/sys/x86/x86/local_apic.c, line 547.

Thread 1 hit Breakpoint 2, lapic_init (addr=addr@entry=4276092928) at <path-to-freebsd-source>/sys/x86/x86/local_apic.c:547
547             KASSERT((cpu_feature & CPUID_TSC) != 0 && tsc_freq != 0,

Thread 1 hit Breakpoint 1, clockcalib (clk=0xffffffff80957a20 <cb_lapic_getcount>, clkname=0xffffffff80983fa9 "lapic")
    at <path-to-freebsd-source>/sys/kern/subr_clockcalib.c:114
114                             printf("Statistical %s calibration failed!  "
(gdb) bt
#0  clockcalib (clk=0xffffffff80957a20 <cb_lapic_getcount>, clkname=0xffffffff80983fa9 "lapic")
    at <path-to-freebsd-source>/sys/kern/subr_clockcalib.c:114
#1  0xffffffff80955f8d in lapic_calibrate_initcount (la=<optimized out>) at <path-to-freebsd-source>/sys/x86/x86/local_apic.c:957
#2  lapic_calibrate_timer () at <path-to-freebsd-source>/sys/x86/x86/local_apic.c:849
#3  0xffffffff808f059a in cpu_initclocks () at <path-to-freebsd-source>/sys/x86/isa/clock.c:419
#4  0xffffffff80542300 in initclocks (dummy=<optimized out>) at <path-to-freebsd-source>/sys/kern/kern_clock.c:424
#5  0xffffffff8053dfe4 in mi_startup () at <path-to-freebsd-source>/sys/kern/init_main.c:323
#6  0xffffffff809140bc in start_kernel ()
#7  0x0000003f0000003f in ?? ()
#8  0x0000000000000000 in ?? ()
```

Looking at the definitions of `lapic_calibrate_timer` and
`lapic_calibrate_initcount`, they're very short.

```c
lapic_calibrate_timer(void)
{
        struct lapic *la;
        register_t intr;

#ifdef DEV_ATPIC
        /* Fail if the local APIC is not present. */
        if (!x2apic_mode && lapic_map == NULL)
                return;
#endif

        intr = intr_disable();
        la = &lapics[lapic_id()];

        lapic_calibrate_initcount(la);

        intr_restore(intr);

        if (lapic_timer_tsc_deadline && bootverbose) {
                printf("lapic: deadline tsc mode, Frequency %ju Hz\n",
                    (uintmax_t)tsc_freq);
        }
}
```
{: file='sys/x86/x86/local_apic.c' }

```c
static void
lapic_calibrate_initcount(struct lapic *la)
{
        uint64_t freq;

        if (lapic_calibrate_initcount_cpuid_vm())
                goto done;

        /* Calibrate the APIC timer frequency. */
        lapic_timer_set_divisor(2);
        lapic_timer_oneshot_nointr(la, APIC_TIMER_MAX_COUNT);
        fpu_kern_enter(curthread, NULL, FPU_KERN_NOCTX);
        freq = clockcalib(cb_lapic_getcount, "lapic");
        fpu_kern_leave(curthread, NULL);

        /* Pick a different divisor if necessary. */
        lapic_timer_divisor = 2;
        do {
                if (freq * 2 / lapic_timer_divisor < APIC_TIMER_MAX_COUNT)
                        break;
                lapic_timer_divisor <<= 1;
        } while (lapic_timer_divisor <= 128);
        if (lapic_timer_divisor > 128)
                panic("lapic: Divisor too big");
        count_freq = freq * 2 / lapic_timer_divisor;
done:
        if (bootverbose) {
                printf("lapic: Divisor %lu, Frequency %lu Hz\n",
                    lapic_timer_divisor, count_freq);
        }
}
```
{: file='sys/x86/x86/local_apic.c' }

However, before analyzing these, something caught my eye in the nearby function
`lapic_calibrate_initcount_cpuid_vm`.

```c
	if (vm_guest == VM_GUEST_NO)
		return (false);
	if (hv_high < 0x40000010)
		return (false);
	do_cpuid(0x40000010, regs);
	freq = (uint64_t)(regs[1]) * 1000;
```
{: file='sys/x86/x86/local_apic.c' }

This is extremely similar logic to what we saw in `tsc_freq_cpuid_vm`. It's not
relevant for this particular run, as `vm_guest` is 1 at this point in execution
while `VM_GUEST_NO` is 0. This makes me wonder what the Firecracker port of
FreeBSD is doing at that function, which is admittedly something I should have
checked earlier.

However, before that, I was on call with Tom asking about something I observed
regarding the values I saw when analyzing some variables at the `clockcalib`
breakpoint.

```terminal
(gdb) p t1
$1 = 1000001
(gdb) p tc->tc_frequency
$2 = 1000000
```

It's quite odd that `t1` is exactly one tick greater than the timecounter
frequency value when the calibration failure message is printed. He wanted to
know what the timer name was, so I printed the entire structure.

```terminal
(gdb) p *tc
$3 = {tc_get_timecount = 0xffffffff805d3200 <dummy_get_timecount>, tc_poll_pps = 0x0, tc_counter_mask = 4294967295, tc_frequency = 1000000,
  tc_name = 0xffffffff809870d9 "dummy", tc_quality = -1000000, tc_flags = 0, tc_priv = 0x0, tc_next = 0x0, tc_fill_vdso_timehands = 0x0,
  tc_fill_vdso_timehands32 = 0x0}
```

This changes everything we know about the project. The printed message reports
a local APIC issue because of the function parameter, but our actual issue is
that the timer is never set to anything else but the dummy timer. If we look
at assignments to the global `timecounter` variable, the only relevant
assignments are the initial value and the one in the definition of `tc_init`
in `sys/kern/kern_tc.c`.

If we try to set a breakpoint on `tc_init`, it's never tripped. Hence, the next
step to take in this project is to get `tc_init` to trip.


## Firecracker Detour

I created a psuedo-guide on how to debug Firecracker guests with GDB at
</posts/firecracker-guest-debugging-with-gdb/>, as I was frustrated with the
poor official documentation on the matter. My goal was to use FreeBSD's
Firecracker port as another analysis vector. Unfortunately, the port is
currently broken.

```terminal
GDB: no debug ports present
KDB: debugger backends: ddb
KDB: current backend: ddb
---<<BOOT>>---
panic: mptable_walk_table: Unknown MP Config Entry 68

cpuid = 0
time = 1
KDB: stack backtrace:
??() at 0xffffffff803a15db/frame 0xffffffff81403dd0
??() at 0xffffffff805c3b26/frame 0xffffffff81403f00
??() at 0xffffffff805c39e3/frame 0xffffffff81403f60
??() at 0xffffffff80966551/frame 0xffffffff81403f80
??() at 0xffffffff80965841/frame 0xffffffff81403fa0
??() at 0xffffffff80543b54/frame 0xffffffff81403ff0
KDB: enter: panic
[ thread pid 0 tid 0 ]
Stopped at      0xffffffff80612f13
```

Trying to set a breakpoint on `mptable_walk_table` gives me an address of
`0x809659af`, while the panic is happening at address `0xffffffff80612f13` in
the run above, with all backtrace addresses being in that ballpark. Given this,
I tried stalling `mptable_walk_table` when `*entry` is 68

```c
/*
 * Call the handler routine for each entry in the MP config base table.
 */
static void
mptable_walk_table(mptable_entry_handler *handler, void *arg)
{
	u_int i;
	u_char *entry;
	
	entry = (u_char *)(mpct + 1);
	
	printf("mpct + 1: %p\n", mpct + 1);
	printf("entry: %d\n", *entry);
	if (*entry == 68) {
		while (true) {

		}
	}
	
	for (i = 0; i < mpct->entry_count; i++) {
		switch (*entry) {
		case MPCT_ENTRY_PROCESSOR:
		case MPCT_ENTRY_IOAPIC:
		case MPCT_ENTRY_BUS:
		case MPCT_ENTRY_INT:
		case MPCT_ENTRY_LOCAL_INT:
			break;
		default:
			panic("%s: Unknown MP Config Entry %d\n", __func__,
			    (int)*entry);
		}
		handler(entry, arg);
		entry += basetable_entry_types[*entry].length;
	}
}
```
{: file='sys/x86/x86/mptable.c' .nolineno }

but this resulted in the same panic.

```terminal
GDB: no debug ports present
KDB: debugger backends: ddb
KDB: current backend: ddb
---<<BOOT>>---
mpct + 1: 0xffffffff8009fc3c
entry: 0
panic: mptable_walk_table: Unknown MP Config Entry 68

cpuid = 0
time = 1
KDB: stack backtrace:
??() at 0xffffffff803a15db/frame 0xffffffff81403dd0
??() at 0xffffffff805c3b26/frame 0xffffffff81403f00
??() at 0xffffffff805c39e3/frame 0xffffffff81403f60
??() at 0xffffffff80966551/frame 0xffffffff81403f80
??() at 0xffffffff80965841/frame 0xffffffff81403fa0
??() at 0xffffffff80543b54/frame 0xffffffff81403ff0
KDB: enter: panic
[ thread pid 0 tid 0 ]
Stopped at      0xffffffff80612f13
```

Later, [Colin Percival](mailto:cperciva@tarsnap.com) pointed out that the value of
`*entry` changes non-atomically in the loop, but I haven't tested anything
with this knowledge yet.

I figured to just stall the function in all cases at that point.

```c
/*
 * Call the handler routine for each entry in the MP config base table.
 */
static void
mptable_walk_table(mptable_entry_handler *handler, void *arg)
{
	u_int i;
	u_char *entry;
	
	entry = (u_char *)(mpct + 1);
	
	printf("mpct + 1: %p\n", mpct + 1);
	printf("entry: %d\n", *entry);
  while (true) {

  }
	
	for (i = 0; i < mpct->entry_count; i++) {
		switch (*entry) {
		case MPCT_ENTRY_PROCESSOR:
		case MPCT_ENTRY_IOAPIC:
		case MPCT_ENTRY_BUS:
		case MPCT_ENTRY_INT:
		case MPCT_ENTRY_LOCAL_INT:
			break;
		default:
			panic("%s: Unknown MP Config Entry %d\n", __func__,
			    (int)*entry);
		}
		handler(entry, arg);
		entry += basetable_entry_types[*entry].length;
	}
}
```
{: file='sys/x86/x86/mptable.c' .nolineno }

This got us to output

```terminal
GDB: no debug ports present
KDB: debugger backends: ddb
KDB: current backend: ddb
---<<BOOT>>---
mpct + 1: 0xffffffff8009fc3c
entry: 0
```

The strange part to me is that, while the function is stalled in the infinite
loop, GDB recognizes that we're in `mptable_walk_table`. However, the assembly
view and breakpoint tell us that the function is located at a lower address
`0x809659af`, while `rip` has an address `0xffffffff80966240`. If we scroll in
the assembly view, we see that the offset seems to be `0xffffffff00000000`.
I put that to the test by recompiling without the infinite loop.

```terminal
(gdb) target remote /tmp/gdb.socket
Remote debugging using /tmp/gdb.socket
warning: No executable has been specified and target does not support
determining executable automatically.  Try using the "file" command.
0x0000000000921000 in ?? ()
(gdb) add-symbol-file kernel
add symbol table from file "kernel"
(y or n) y
warning: A handler for the OS ABI "FreeBSD" is not built into this configuration
of GDB.  Attempting to continue with the default i386:x86-64 settings.

Reading symbols from kernel...
warning: remote target does not support file transfer, attempting to access files from local filesystem.
Reading symbols from <directory-containing-kernel>/kernel.debug...
(gdb) b mptable_walk_table
Breakpoint 1 at 0x809659bf: mptable_walk_table. (7 locations)
(gdb) b *(0x809659bf + 0xffffffff00000000)
Note: breakpoint 1 also set at pc 0x809659bf.
Breakpoint 2 at 0x809659bf: file <path-to-freebsd-source>/sys/x86/x86/mptable.c, line 481.
(gdb) c
Continuing.
```

Unfortunately, I still got the same kernel panic without tripping the
breakpoint. I put the infinite loop back in and printed everything I could
think of about `mpct`.

```terminal
(gdb) p mpct
$1 = (mpcth_t) 0x8009fc10
(gdb) p mpct.signature
$2 = "PCMP"
(gdb) p mpct.base_table_length
$3 = 288
(gdb) p mpct.spec_rev
$4 = 4 '\004'
(gdb) p mpct.checksum
$5 = 69 'E'
(gdb) p mpct.oem_id
$6 = "FC      "
(gdb) p mpct.product_id
$7 = '0' <repeats 12 times>
(gdb) p mpct.oem_table_pointer
$8 = 0
(gdb) p mpct.oem_table_size
$9 = 0
(gdb) p mpct.entry_count
$10 = 30
(gdb) p/x mpct.apic_address
$11 = 0xfee00000
(gdb) p mpct.extended_table_length
$12 = 0
(gdb) p mpct.extended_table_checksum
$13 = 0 '\000'
(gdb) p mpct.reserved
$14 = 0 '\000'
```

I don't know what `mpct` is, and Colin didn't address it. Besides the
non-atomic variable `entry` from earlier, this seems to be the point my
Firecracker journey ends in this project.
