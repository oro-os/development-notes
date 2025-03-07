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

