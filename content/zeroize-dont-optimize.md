+++
title = "Zeroize, Don't Optimize: Why Compilers Delete Your Security Code"
date = 2025-09-14

[extra]
summary = "A look at why compilers optimize away memory zeroing, and how Rust's volatile writes solve it."
+++

## The problem

You finish processing a secret — a private key, a password, a token. You zero the memory. You move on. The secret is gone, right?

Not necessarily. The compiler saw you write zeros to a buffer that's never read again. From its perspective, that write is dead code. It removes it. Your secret sits in memory until the OS reclaims the page, which could be never.

This isn't theoretical. It's a well-documented issue across C, C++, and any language where the compiler has freedom to eliminate dead stores.

## Why it happens

Compilers optimize for observable behavior. If a write has no effect on the program's output — no subsequent read, no side effect — the compiler considers it redundant and removes it. This is correct from a performance standpoint. It's catastrophic from a security standpoint.

The memory contents persist. They can be read through:

- Core dumps
- Swap files
- `/proc/<pid>/mem` on Linux
- Cold boot attacks
- Any memory inspection tool

## The volatile solution

The standard fix is to use a volatile write. Volatile tells the compiler: this write has side effects you can't see. Don't optimize it away.

In Rust, `core::ptr::write_volatile` does this:

```rust
use core::ptr;

fn secure_zero(buf: &mut [u8]) {
    for byte in buf.iter_mut() {
        unsafe { ptr::write_volatile(byte, 0); }
    }
}
```

The compiler can't remove the volatile writes. The buffer gets zeroed. The secret is gone.

## What about the compiler fence?

Volatile prevents the compiler from removing individual writes. But compilers can also reorder operations. A memory fence ensures the zeroing happens in the right order relative to other operations.

```rust
use core::sync::atomic::{fence, Ordering};

fn secure_zero_fenced(buf: &mut [u8]) {
    for byte in buf.iter_mut() {
        unsafe { ptr::write_volatile(byte, 0); }
    }
    fence(Ordering::SeqCst);
}
```

The `fence` call acts as a compiler barrier — the compiler won't reorder the zeroing past it.

## What memguard-rs does

The crate wraps all of this into a type system. You don't think about volatile writes or fences. You wrap your secret in a `GuardedVec<u8>`, and when it drops, the memory is zeroed through volatile writes with a compiler fence. The zeroization is guaranteed by the type's `Drop` implementation, and the compiler can't optimize it away.

The next post will cover `mlock` and why preventing swap is harder than it sounds.
