A few times now I've been bitten by the following:

```rust
fn pick_next_thread_and_switch() -> ! {
    let mut sched = get_scheduler().lock();
	let next_thread = sched.next();
	crate::arch::run_thread(next_thread); // -> !
}
```

At least so far, the interrupt design is such that `run_thread` here is marked `-> !`, meaning it'll never return (on this same codepath). Put simply, an interrupt event starts a whole "new" Rust program, where the program "state" is stored in global variables that are accessed in a rather volatile manner (that's oversimplifying but you get the point).

The problem here, however, is `sched` which, to the astute eye, can be identified as a _mutex guard_ - a structure returned by a `lock()` call that looks something like `MutexGuard<'a>`, where `'a` is the lifetime of the originating lock itself.

One typically relies on dropping behavior of Rust to automatically clean up the `MutexGuard`, which will implement `Drop` and release the lock upon being dropped (typically an atomic store on the original lock object).

However, in the case of `-> !`, Rust opts not to clean up stack value before the never-returning call. I suppose this makes sense, but it is surprising behavior in our particular case, since in most cases it means "this is the end of the program", but in our case, it means "this is the end of this _particular line of execution_". Rust isn't really built for interrupt-based computing, and we're skirting around undefined behavior rules, I'm sure.

I would personally love to see an optional lint that detects this and forces you to ignore it on a case-by-case basis. Maybe even a clippy lint would suffice.