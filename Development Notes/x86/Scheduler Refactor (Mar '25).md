These are notes surrounding the March 2025 refactor of the scheduler, as well as the x86_64 interrupt system (at once). The development throughout the refactor includes testing on QEMU, VMWare Workstation 17, VirtualBox 7.1, and a bare metal Intel machine. Unfortunately I do not have a bare metal AMD machine to test on readily aside from my own, main box.

## \[7 March, 2025]: APIC Troubles

On VBox and bare metal, secondaries are not receiving their timer interrupts. Primary cores do in all environments. VMWare and QEMU are fine. I believe it's because APIC initialization is not robustly and completely performed on secondaries, since they're being booted from scratch (16-bit mode). Trying to figure out why.

It's worth noting that we do not use the 8259 PIC on x86_64, and disable it globally.

> Because the system boots in a PC/AT compatible mode (PIC mode or virtual wire mode), 0x01 should be written to the IMCR ([[Intel MultiProcessor Specification (1997).pdf#page=27&offset=-9,797,0|Intel MultiProcessor Specification (1997), 3.6.2.1 PIC Mode]]) (in case it exists) to disconnect the PIC from the local APIC’s LINT0 INTI (see Figure 2.4). In case the IMCR is not available, all 16 PIC interrupts should be masked.

[[Interrupt Handling using the x86 APIC.pdf#page=26&selection=12,0,19,62|Interrupt Handling using the x86 APIC, page 26]]

I'm not sure if this is required if the 8259 is disabled. It's also not clear if the 8259 needs to be disabled on all cores. Doing so on secondaries doesn't appear to make a difference, and I've heard from the \#osdev Libera folks before that doing so is a 'global' thing anyway.

We currently disable it globally by masking of all interrupts, though disconnecting it via the IMCR is new. I should test that.

Ah. See [[Intel MultiProcessor Specification (1997).pdf#page=28&selection=60,0,60,20|Intel MultiProcessor Specification (1997), page 28]]. In PIC mode, interrupts go directly to the BSP. Nowhere else. That might be why this is happening.

> To access the IMCR, write a value of 70h to I/O port 22h, which selects the IMCR. Then write the data to I/O port 23h. The power-on default value is zero, which connects the NMI and 8259 INTR lines directly to the BSP. Writing a value of 01h forces the NMI and 8259 INTR signals to pass through the APIC

[[Intel MultiProcessor Specification (1997).pdf#page=28&selection=63,31,66,50|Intel MultiProcessor Specification (1997), page 28]]

So then the question becomes how to detect IMCR, and the logic is basically

```
# Mask off interrupts.
OUTB(21h, 0xFF)
OUTB(A1h, 0xFF)
IF IMCR:
	# Disconnect entirely.
    OUTB(22h, 70h)
	OUTB(23h, 1)
```

This is confirmed later on:

> The operating system should not write to the IMCR unless the IMCRP bit is set.

[[Intel MultiProcessor Specification (1997).pdf#page=76&selection=46,19,47,7|Intel MultiProcessor Specification (1997), page 76]]

IMCRP bit is here: [[Intel MultiProcessor Specification (1997).pdf#page=90&selection=97,0,102,11|Intel MultiProcessor Specification (1997), page 90]]

The "floating pointer structure" is defined in [[Intel MultiProcessor Specification (1997).pdf#page=89&selection=17,0,22,64|Intel MultiProcessor Specification (1997), page 89]] and finding it is described here: [[Intel MultiProcessor Specification (1997).pdf#page=38&selection=8,1,27,57|Intel MultiProcessor Specification (1997), page 38]]

Meh. I hate memory probing, but I guess it's going to be necessary. Looks like we're searching for `_MP_` in that area (in sequential order): [[Intel MultiProcessor Specification (1997).pdf#page=39&selection=58,0,58,41|Intel MultiProcessor Specification (1997), page 39]]. That marks the beginning of the floating point table.

I'll do this later. Added a `// TODO` for now.

Beginning to think I'm just crossing APIC operation modes when I shouldn't be. Need to figure out the difference between LIDT vectors and the LAPIC vectors.

A little while later (after consulting ChatGPT's Deep Research) I discovered that on some strict systems, the spurious interrupt register _must_ be set up before the timer is set up. That was my mistake - I was setting up the spurious interrupts after the timer initialization. Works in VBox, testing now on bare metal... Yep! Works.

## \[8 March, 2025] Interrupt Design

I was originally wanting a function call-like interface such that the kernel calls a `run_context` arch-specific trait function with the arch's native thread handle but I realized that'd mean the archs would always have to context switch back to the kernel, which is a huge overhead for a code nicety (effectively double the context switching cost, guaranteed - 4x switches per user-thread switch (user store - kernel restore - kernel store - user restore)).

So I'm opting for a middle ground. Currently there are individual callbacks for each type of event that could occur, causing a lot of duplication, but I think a single "event" point that is funneled to in the kernel whereby the `run_context` is `-> !` and the event entry point doesn't assume anything about the state of the system is acceptable, as that means the kernel's execution can just start from the stack top each time it's invoked.

This also means that the kernel's state must not refer to anything on the stack. This will have to be documented by a `# Safety` note. This is because the stack will be munged any time an interrupt happens, and thus interrupts cannot be enabled during kernel execution (this is already the case). They're only enabled when the stack is no longer being used.

The "funnel" will then have an enum for the event that occurred. I think I can achieve a clean interrupt design from this, as I've played with this already with the current implementation of the "default" (early-boot) interrupt system that is used prior to the older kernel-specific (userspace handling) interrupt system.

It also means that any exceptions (vectors < 32) can have stub-specific assembly to check the code segment upon entry for its DPL (`cs & 3 == 0` means it's an exception from Ring 0, i.e. the kernel faulted) and jump to a core dump panic handler instead.

Similar to the default handlers now, an error code will be pushed if the trampoline is _not_ an error-code interrupt, followed by the IV, followed by the register state, all pushed to the shadow stack.

The RSP is then stored in `rdi` (first argument in a system-v call) and execution is `jmp`'d to an `extern "C"` function that accepts a `*const UnsafeCell<StackFrame>`. The same is done for system calls, whereby the IV is `!0`.

A note about system calls: the same stack frame is passed, but not all registers are populated. The kernel is specified as to only resuming a context with a system call response if it was switched away from due to a syscall. In such a case the architecture can elide certain registers that aren't used as part of the system call API, and can simply XOR them on a `sysret` in order to clear out any potentially sensitive data.

# \[9 March, 2025] Fun with `global_asm!` linkage

I ran into a funny issue today with the new interrupt system (which, I dare say, is about 1000x better than what was there before).

Everything worked great - more or less first try - on all four of the environments... in debug mode. Release mode crashed with a page fault. Which is weird, because there was no reason it should have been doing that environment-wise. The fact it happened on all four (from what I could tell) was reassuring - namely, QEMU, which if QEMU is faulting it typically means a logic error instead of a machine quirk, which is good news as it means I can debug this in a reliable fashion.

Further, I have invested considerable time so far in the debug tooling, which ended up letting me solve this in about **5 minutes**.

The symptoms were this:

- The timer expiration interrupt is fired (expected)
- The trampoline pushes a synthetic error code (`0`) as well as the vector number (`0x20`) onto the stack and jumps to `_oro_isr_common_novec` (common ISR handler, no vector register preservation variant).
- CPU immediately jumps to ISR handler `0x0e` (page fault) with a `cr2` of the instruction we just came from, and an error code of `0x11` - `0x1` indicating that the page _is_ present (and thus the fault is due to access flags rather than missing memory), and `0x10` indicating that the fault was an instruction fetch (something was trying to be executed, rather than a normal memory store/load).

Very interesting. This is just a bog-standard `global_asm!` macro invocation that is defining this stuff (albeit it's really a `global_asm!(include_str!("./isr64.S"))`, but alas, that shouldn't matter). There should be no reason this faults.

Without good tooling, this would require some rather tedious debug printing, as GDB does not allow direct physical memory walking. I'd have to find the kernel's stored linear offset, manually fetch `cr3` and add that offset, read that memory, decode the values by hand, etc.

Luckily, I have a whole [debug suite](https://github.com/oro-os/kernel/blob/master/) worth of Python-based GDB tooling - most notably, a full QEMU monitor link and frontend commands that intertwine GDB and QEMU monitor commands into higher level debug functions.

Since for page faults the faulting memory address is stored in `cr2`, I can simply run the Oro debug utility `oro tt v` - a 't'able/'t'ranslation utility for 'v'irtual addresses. It walks the page tables for me, spitting out each of the level-specific flags and their meanings, and then cross-references that with QEMU's monitor command (which only does the translation, not providing any useful information about the walk itself) to see if we arrive at the same result.

```
(oro-gdb) bt
#0  0xffffffff8000114d in _oro_isr_handler_14_novec ()
#1  0x0000000000000011 in ?? ()
#2  0xffffffff800217ff in _oro_isr_common_exc_novec ()
#3  0x0000000000000008 in ?? ()
#4  0x0000000000010046 in ?? ()
#5  0xffff80ffffffefc8 in ?? ()
#6  0x0000000000000010 in ?? ()
#7  0x00007f8000000000 in ?? ()
#8  0x0000000000000020 in ?? ()
#9  0x0000000000000000 in ?? ()
(oro-gdb) print $cr2
$6 = -2147346433
(oro-gdb) print /x $cr2
$7 = 0xffffffff800217ff
(oro-gdb) or tt v $cr2
oro tt: cannot check CPUID; PSE and 1G page support cannot be verified 
oro tt: cannot check EFER.LME; assuming long mode is enabled 
oro tt: CR3=0x0000000000053000
oro tt: CR4.LA57=False (5-level paging is disabled)
oro tt: L4 index: 511
oro tt: L4                = 0x0000000000055023
oro tt:   .P              = 1 (present)
oro tt:   .RW             = 1 (read/write)
oro tt:   .US             = 0 (supervisor)
oro tt:   .PWT            = 0 (write-back)
oro tt:   .PCD            = 0 (uncacheable)
oro tt:   .A              = 1 (accessed)
oro tt:   .AVL[6]         = 0b0
oro tt:   .PS             = 0 (table entry)
oro tt:   .AVL[11:8]      = 0b0000 (0)
oro tt:   .PHYS           = 0x0000000000055000
oro tt:   .AVL[62:52]     = 0b00000000000 (0)
oro tt:   .XD             = 0 (executable)
oro tt: L3 index: 510
oro tt: L3                = 0x0000000000056023
oro tt:   .P              = 1 (present)
oro tt:   .RW             = 1 (read/write)
oro tt:   .US             = 0 (supervisor)
oro tt:   .PWT            = 0 (write-back)
oro tt:   .PCD            = 0 (uncacheable)
oro tt:   .A              = 1 (accessed)
oro tt:   .AVL[6]         = 0b0
oro tt:   .PS             = 0 (table entry)
oro tt:   .AVL[11:8]      = 0b0000 (0)
oro tt:   .PHYS           = 0x0000000000056000
oro tt:   .AVL[62:52]     = 0b00000000000 (0)
oro tt:   .XD             = 0 (executable)
oro tt: L2 index: 0
oro tt: L2                = 0x0000000000057023
oro tt:   .P              = 1 (present)
oro tt:   .RW             = 1 (read/write)
oro tt:   .US             = 0 (supervisor)
oro tt:   .PWT            = 0 (write-back)
oro tt:   .PCD            = 0 (uncacheable)
oro tt:   .A              = 1 (accessed)
oro tt:   .AVL[6]         = 0b0
oro tt:   .PS             = 0 (table entry)
oro tt:   .AVL[11:8]      = 0b0000 (0)
oro tt:   .PHYS           = 0x0000000000057000
oro tt:   .AVL[62:52]     = 0b00000000000 (0)
oro tt:   .XD             = 0 (executable)
oro tt: L1 index: 33
oro tt: L1                = 0x8000000000078121
oro tt:   .P              = 1 (present)
oro tt:   .RW             = 0 (read-only)
oro tt:   .US             = 0 (supervisor)
oro tt:   .PWT            = 0 (write-back)
oro tt:   .PCD            = 0 (uncacheable)
oro tt:   .A              = 1 (accessed)
oro tt:   .D              = 0 (not dirty)
oro tt:   .PAT            = 0
oro tt:   .G              = 1 (global)
oro tt:   .AVL[11:9]      = 0b000 (0)
oro tt:   .PHYS           = 0x0000000000078000
oro tt:   .AVL[58:52]     = 0b0000000 (0)
oro tt:   .PK             = 0x0 (0)
oro tt:   .XD             = 1 (execute disable)
oro tt: physical address: 0x00000000000787ff
oro tt: verifying manual walk with QEMU translation...
oro tt: QEMU translation: 0x00000000000787ff
oro tt: QEMU translation matches manual walk - OK!
```

Hmm so indeed the page is there, it's read-only, but... oh.

```
oro tt:   .XD             = 1 (execute disable)
```

Well that's strange. It's machine code, it should absolutely not be marked as "don't execute".

I ran the kernel image against `readelf -a | grep` to get the symbol address:

```
401: ffffffff8002177e 0 NOTYPE GLOBAL DEFAULT 3 _oro_isr_common_exc_novec
```

and then `readelf -l` to see which program segment it belongs to:

```
Program Headers:
  Type           Offset             VirtAddr           PhysAddr
                 FileSiz            MemSiz              Flags  Align
  LOAD           0x0000000000001000 0xffffffff80000000 0xffffffff80000000
                 0x0000000000000100 0x0000000000000100  R      0x1000
  LOAD           0x0000000000002000 0xffffffff80001000 0xffffffff80001000
                 0x000000000001f845 0x000000000001f845  R E    0x1000
  LOAD           0x0000000000022000 0xffffffff80021000 0xffffffff80021000
                 0x0000000000000abe 0x0000000000000abe  R      0x1000
  LOAD           0x0000000000023000 0xffffffff80022000 0xffffffff80022000
                 0x0000000000002b48 0x0000000000002b48  RW     0x1000

 Section to Segment mapping:
  Segment Sections...
   00     .oro_boot 
   01     .text 
   02     .rodata 
   03     .data .bss .got
```

It takes a second to decode visually, but it's being placed in segment `02` - or rather, the `.rodata` (read only data) segment.

It seemed a little obvious after that what the problem was, though it was something I had never considered - `global_asm!` doesn't make any assumption about which section you want it to place the machine code into. I guess that makes complete sense, but it was definitely surprising.

So, I added this to the top of the `isr.S` file:

```
.section .text.oro_isr64
```

rebuilt, and sure enough that seemed to work - here's the `readelf -a | grep`:

```
401: ffffffff80002595 0 NOTYPE GLOBAL DEFAULT 2 _oro_isr_common_exc_novec
```

and the `readelf -l`:

```
Program Headers:
  Type           Offset             VirtAddr           PhysAddr
                 FileSiz            MemSiz              Flags  Align
  LOAD           0x0000000000001000 0xffffffff80000000 0xffffffff80000000
                 0x0000000000000100 0x0000000000000100  R      0x1000
  LOAD           0x0000000000002000 0xffffffff80001000 0xffffffff80001000
                 0x0000000000020025 0x0000000000020025  R E    0x1000
  LOAD           0x0000000000023000 0xffffffff80022000 0xffffffff80022000
                 0x00000000000002ce 0x00000000000002ce  R      0x1000
  LOAD           0x0000000000024000 0xffffffff80023000 0xffffffff80023000
                 0x0000000000002b48 0x0000000000002b48  RW     0x1000

 Section to Segment mapping:
  Segment Sections...
   00     .oro_boot 
   01     .text 
   02     .rodata 
   03     .data .bss .got
```

And sure enough, firing it up in the VM allowed me to step through it and see that it's able to execute those instructions just fine now. Cool!

This is becoming a very, very frequent trend throughout this project - the time spent on debugging tools is never time I regret spending. Debugging this just to find the `NX` bit would have taken me hours or even days before (and it has, believe me). The more I work on these, the more equipped I feel to take on even the hardest or most obscure kernel bugs I encounter, which is, of course, priceless.

(Update; next day) According to [this comment](https://github.com/rust-lang/rust/issues/138247#issuecomment-2708600901) it might actually be undefined behavior to put section directives in `global_asm!` which is news to me. I also don't see this documented anywhere (and I'm still waiting to hear back). However, this might actually be true - the boot stubs _do_ use `.section .rodata` earlier in the code, which might be affecting it. Might be good to go down a bit of a rabbit hole to determine how all of this is being set up and put together.

# \[9 March, 2025] On to syscalls

Just a quick note. Context switching is now in; user mode is now being switched to. The first fault I got when running the hello world module was, interestingly, the invalid instruction fault. My first inclination was that it was a vector instruction or something, but I had doubts since the Rust toolchain for Oro explicitly disables them for now (though after this refactor they can be enabled).

I had the faulting IP and needed to get the instruction for that address. I figured I could just `nm` it and look for the symbol before the address and disassemble it but I knew that'd take time. The module was also a release module, which means that no symbols would be shown anyway.

The next idea was to open it up in gdb and try that, but I knew that'd be clunky, time consuming (I'm trying to get more efficient as I go along) and probably not ideal.

I asked ChatGPT and it suggested `objdump` and Radare2 (`r2`) as potential solutions. I tried `objdump` but it disassembles in a weird format and the addresses are all file-relative, not load relative (or at least, I didn't spend much time to figure out if there was a way to do that). Grepping for the address seemed to be a pain, and I knew there was probably a better way anyway.

Radare2 was a new suggestion though. I've heard of it and been meaning to try it, but haven't had the need to just yet. The command ChatGPT suggested seemed relatively straightforward, too, and seemed to be load-relative - important, since I only had the loaded virtual address that faulted, and no (easy) way to get the originating file address.

```shell
r2 -c "pd 1 @ 0x401000" -q mybinary
```

Turns out it worked perfectly. The faulting address was `0x200769a`, so I just plugged it in, and...

```shell
$ r2 -c "pd 1 @ 0x200769a" -q $ORO_ROOT_MODULES
0x0200769a      0f05           syscall
```

Seems right, since the refactor removed the old syscall handling code and installer for now. Just didn't expect it to turn into a bad instruction exception. Seems to be that the latest context switching code works as expected!