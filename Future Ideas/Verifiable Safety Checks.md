Right now, if a function is marked as an `unsafe fn`, the kernel has a lint that enforces its documentation has a `# Safety` section. That's fine and well, but the guarantees therein are not machine-readable nor are they verifiable.

I'd like to eventually implement a system of attributes that allow documenting functions as unsafe with specific unsafe markers - `interrupts_disabled`, `thread_not_running`, `instance_not_dead`, etc.

```rust
/// Does a critical thing.
#[safety(interrupts_disabled, "interrupts must be disabled before calling")]
unsafe fn do_critical_thing() { /* ... */ }

/// Sets the value of `CR3`.
#[safety(phys_valid, "the physical address must be valid")]
#[safety(phys_is_pt, "the physical address must point to a valid page table")]
#[safety(page_aligned, "the physical address must be page aligned")]
unsafe fn set_cr3(pt: u64) { /* ... */ }
```

For callers, this would take the place of the `// SAFETY: ...` comment that is part of the current Clippy lint:

```rust
#[unsafe(safety(phys_valid, phys_is_pt, page_aligned))]
unsafe { set_cr3(some_phys); }
```


The `rustc` linter would then do a pass that would check these and emit diagnostics for any mismatches.

Let's say that `set_cr3` also adds a new safety attribute:

```rust
/// Sets the value of `CR3`.
// ..
#[safety(cr3_bits, "CR3 is a bitfield; the lower bits must represent the ASID")]
// ..
unsafe fn set_cr3(pt: u64) { /* ... */ }
```

But our callsite doesn't include `cr3_bits`! The linter would probably output something like

```
error[E0133]: unsatisfied #[safety] requirement: CR3 is a bitfield; the lower bits must represent the ASID
   --> oro-arch-x86_64/src/boot/primary.rs:104:3
    |
104 |         crate::asm::set_cr3(new_pt);
    |         ^^^^^^^^^^^^^^^^^^^^^^^^^^^ 'cr3_bits' is unsatisfied
    |
    = note: for more information, see <https://doc.rust-lang.org/nightly/edition-guide/rust-2024/safety-requirement.html>
    = note: consult the function's documentation for information on how to avoid undefined behavior
note: the safety requirement is defined here
   --> oro-arch-x86_64/src/asm.rs:100:9
    |
100 |     #[safety(cr3_bits, "CR3 is a bitfield; the lower bits must represent the ASID")]
    |              ^^^^^^^^

For more information about this error, try `rustc --explain E0133`
```

Likewise, if a safety requirement is _removed_ from a function, but a callsite still documents it, the compiler will emit a similar (but inverse) error message.
## Taking it further

This could definitely turn into a more rigid checker, though it'd require the ability to add attributes to expressions (which is current unstable: https://github.com/rust-lang/rust/issues/15701

```rust
/// Sets the value of CR3
#[safety(memmap_change, "caller must be ready for a memory map change")]
unsafe fn set_cr3(
	#[safety(phys_valid, "the physical address must be valid")]
	#[safety(phys_is_pt, "the physical address must point to a valid page table")]
	#[safety(page_aligned, "the physical address must be page aligned")]
	pt: u64
) {
    // ...
}
```

Then for a naive start:

```rust
#[unsafe(safety(memmap_change))]
unsafe {
	set_cr3(
		#[unsafe(safety(phys_valid, phys_is_pt, page_aligned))]
		new_pt
	)
}
```

This works, but then could also be extended to be 'attached' to values (though this is probably much easier said than done).

```rust
#[unsafe(safety(return_infers(phys_valid, phys_is_pt, page_aligned)))]
fn get_pt() -> u64 {
	return 0x1234000;
}

let new_pt = get_pt(); // `new_pt` now has 'sticky' safety information.

#[unsafe(safety(memmap_change))] // only thing that's necessary, now.
unsafe { set_cr3(new_pt); } // the rest are inferred.
```

## References

Going about implementing the naive case probably isn't super difficult, though I'm not at all versed in rust compiler internals.

For rustc stuff:

- https://doc.rust-lang.org/nightly/nightly-rustc/rustc_middle/thir/index.html
- https://doc.rust-lang.org/nightly/nightly-rustc/rustc_hir/intravisit/index.html
- https://doc.rust-lang.org/nightly/nightly-rustc/rustc_hir/intravisit/trait.Visitor.html
- https://doc.rust-lang.org/nightly/nightly-rustc/rustc_passes/index.html

And some prior art of these sorts of things being used:

- https://crates.io/crates/dylint