I'm implementing high-precision timing support and the timekeeping subsystem for x86_64. This will be a mostly arch-implemented mechanism whereby the kernel can cross-reference thread arrival times in the scheduler against a monotonically increasing clock.

The problem with going about this using interrupts is that the kernel, as designed, is interrupt-free when performing kernel operations. This includes syscalls, task switching, etc. The side effect of this is that I cannot react to timing interrupts in a surefire way without enabling NMIs, which means that on systems without an IOAPIC, potentially non-maskable exceptions also get routed through the same handler, increasing both latency and the surface area for bugs to occur. Just feels messy, and also isn't a perfect solution for timing either, especially since such mechanisms are not guaranteed to have periodic interrupts.

Further, NMIs can fire within one another - if for some reason an NMI fires while I'm processing an NMI, the handler becomes reentrant and it gets even more complicated to implement correctly. I'm not 100% certain about it, but it's also a possibility that not all targeted architectures even have an interrupt-based timing mechanism, so having a way such that the architecture can keep track of a timestamp (at whatever resolution, as long as it can be converted to "human" timestamps) and that the kernel can consume and use in a more or less easy way is much easier and straightforward to develop against.

I've chosen first to target the HPET, as it appears to be available on most modern systems. I'm sure support for other timekeeping mechanisms for x86 will pop up over time but for now this seems like a reasonable first choice. I'll not be using interrupts, but instead be updating the running timestamp at interrupt/syscall points. As long as the kernel only requires that a "kernel timeslice" has an up-to-date timestamp at time of entry, I think that affords a great deal of flexibility for architectures to keep that time however they need.

I think it'll be spec'd like so:

```rust
trait CoreHandle {
	/// A single 'instant' in time; used only for comparison of
	/// timestamps, etc.
	type Instant: oro_kernel::time::Instant;
	
    /// Returns the current timestamp of the system.
	///
	/// This timestamp **must** be up to date at least at kernel event handler
	/// entry, and must always increase, never decrease.
	///
	/// This timestamp **does not** need to be the same clock source
	/// across all cores; each core may use its own clock source,
	/// as long as all other requirements are upheld.
	///
	/// This function may return the same timestamp across multiple calls
	/// within a single kernel context switch; it must be updated whenever
	/// a kernel event occurs (interrupt, system call, etc.), at the latest
	/// upon first call to this function within the kernel timeslice.
	fn now(&self) -> Self::Instant;
}
```

## HPET Links

Some interesting HPET specification links:

- [[Intel HPET Specification 1.0a (2004).pdf#page=10&selection=12,0,12,17|Register Overview]]
- [[Intel HPET Specification 1.0a (2004).pdf#page=30&selection=8,0,9,42|HPET ACPI Table 2.0]]

## Configuration Notes

I've confirmed that the HPET table gives me a good address based on the ACPICA definitions but it's very, very strange that the HPET table specified in [[Intel HPET Specification 1.0a (2004).pdf]] is not the same as what's defined by ACPICA. Further, the table in the `acpi` crate (https://docs.rs/acpi/latest/acpi/hpet/struct.HpetInfo.html) does not match, either.

Nonetheless, I get meaningful capabilities and ID information from the register block on each of the environments I tested on.

## Timing

Our instant type uses nano-second precision. This requires a divisor value for the ticks to get the number of nanoseconds.

> Main Counter Tick Period: This read-only field indicates the period at which the counter increments in femptoseconds (10^-15 seconds). A value of 0 in this field is not permitted. The value in this field must be less than or equal to 05F5E100h (10^8 femptoseconds = 100 nanoseconds). The resolution must be in femtoseconds (rather than picoseconds) in order to achieve a resolution of 50 ppm.

[[Intel HPET Specification 1.0a (2004).pdf#page=11&selection=80,0,87,4|Intel HPET Specification 1.0a (2004), page 11]]

So, our `tick_rate` (above) is `femtoseconds / tick`. We want `ticks / x = nanoseconds`. If my math is correct, that's `x = 10^6 / femtosecond_tick_rate`. Unfortunately, that means that there is now a branch; if the tick rate is less than a nanosecond, a division must occur. However if the tick rate is more than a nanosecond, a multiplication must occur. Using floating point here is out of the question.

So instead, I will actually multiply the timer by the femtosecond rate and use that for instants, with a constant divider. Even with femtoseconds, storage in a `u128` means that it'll never overflow - `2^128 femtoseconds` is still `7.8*10^5` the age of the universe. Neat!

The next problem. Division in the kernel is banned. I could make an exception, but where's the fun in that? (We use modulo everywhere but why not overengineer over a cup of coffee?)

So, the new goal is to multiply the time value not just by the femtoseconds-per-tick rate, but by a magic value based on that rate, such that bitwise right shifting the time value by `s` gives us the nanoseconds value.

I pulled my _Hackers Delight_ (ISBN-13 `978-0321842688`) off the shelf and figured out that if I choose a bit width that is sufficiently wide (`s = log2(1000000) ~= 20`) I can compute a magic number such that `ns = ts >> L = (fs * M) / 1000000`.

I need a larger datatype to do this, but luckily the femtoseconds-per-tick is never larger than a 32-bit value, so we can easily use a 64-bit value for this, since we need `L = N + s` bits of space to perform the magic calculation. Therefore, for a 32-bit value, we'd need `32 + 20 = 52` bits.

We can thus perform `M = ceil(2^L / 1000000)` at compile time, store `femtoseconds_per_tick * M` to get the magic value once at runtime, and then do `ticks * magic` to get the timestamp.

We arrived here:

```rust
pub const fn calculate_fsns_magic(femtoseconds_per_tick: FsNsInt) -> u128 {
	const M: u128 = const { 2u128.pow(FS_NS_SHIFT).div_ceil(1_000_000) };
	(femtoseconds_per_tick as u128) * M
}
```

Multiplying the result of that function against the tick count coming from the HPET gives us a value that we can then right shift by `L` (52 in this case) to arrive at a number of nanoseconds with practically perfect accuracy.
