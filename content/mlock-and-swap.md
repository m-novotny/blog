+++
title = "mlock and the Swap Problem"
date = 2025-10-02

[extra]
summary = "Why locking memory pages is harder than calling mlock, and what happens when the OS doesn't cooperate."
+++

## The threat

Even if you zeroize on drop, your secret can hit swap before you get the chance. The kernel's memory manager pages inactive memory to disk. If your secret is in a page that gets swapped out, it persists on disk — potentially long after your process exits.

Disk is harder to clean than RAM. You can zero a buffer in memory. You can't reliably zero specific sectors of a swap partition.

## mlock

`mlock(2)` tells the kernel: don't swap this page. Pin it in RAM.

```rust
use libc::{mlock, munlock};

unsafe fn lock_memory(ptr: *const u8, len: usize) -> Result<(), i32> {
    let ret = mlock(ptr as *const c_void, len);
    if ret == 0 { Ok(()) } else { Err(errno()) }
}
```

Simple enough. But there are limits.

## RLIMIT_MEMLOCK

Linux caps how much memory a process can lock via `RLIMIT_MEMLOCK`. The default is often 64KB — barely enough for a few keys. You can raise it with `ulimit -l` or by setting the limit in `/etc/security/limits.conf`, but that requires root or appropriate capabilities.

On macOS, the limit is similarly restrictive by default.

If you exceed the limit, `mlock` fails with `ENOMEM`. Your secret is not locked. If you don't check the return value, you silently proceed with swappable memory — worse than not trying, because you've created a false sense of security.

## VirtualLock on Windows

Windows uses `VirtualLock` instead of `mlock`:

```rust
#[cfg(windows)]
unsafe fn lock_memory(ptr: *const u8, len: usize) -> Result<(), u32> {
    use windows::Win32::System::Memory::{VirtualLock, VirtualUnlock};
    
    if VirtualLock(ptr as *const c_void, len).as_bool() {
        Ok(())
    } else {
        Err(GetLastError())
    }
}
```

Same concept, different API. Same limits — the working set minimum must be set, and there are per-process quotas.

## What memguard-rs does

The crate attempts `mlock` (or `VirtualLock` on Windows) on allocation. If it fails, it logs a warning and continues — the memory is still zeroized on drop, it's just not swap-resistant. The alternative — failing the allocation entirely — would break too many use cases where mlock limits can't be raised (containers, unprivileged environments).

The honest answer is that mlock is best-effort. The crate can't guarantee swap resistance in all environments. What it can guarantee is zeroization on drop, and that alone eliminates most of the attack surface.

## Tradeoffs

Locking too much memory degrades system performance — the kernel can't reclaim those pages under pressure. The crate locks at page granularity and only for the lifetime of the guarded allocation. Users who need to lock large buffers should raise `RLIMIT_MEMLOCK` and monitor `vmstat` for swap activity.
