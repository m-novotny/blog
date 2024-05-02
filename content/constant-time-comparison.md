+++
title = "Constant-Time Comparison: Why == Leaks"
date = 2025-11-19

[extra]
summary = "How branching comparisons leak secret values through timing, and what to do about it."
+++

## The timing side channel

Comparing two values with `==` seems innocent. But `==` short-circuits — it returns false as soon as it finds a mismatched byte. The time it takes reveals how many bytes matched.

An attacker measures comparison time, iterates through possible values, and watches the timing change. Each time a byte matches, the comparison takes slightly longer. One byte at a time, the secret leaks.

This isn't theoretical. Padding oracle attacks, timing attacks on JWT verification, and HMAC comparison leaks have all been exploited in the wild.

## Constant-time comparison

The fix: compare every byte, every time, regardless of early mismatches. Accumulate a difference, return at the end.

```rust
fn ct_eq(a: &[u8], b: &[u8]) -> bool {
    if a.len() != b.len() {
        return false;
    }
    let mut diff: u8 = 0;
    for (x, y) in a.iter().zip(b.iter()) {
        diff |= x ^ y;
    }
    diff == 0
}
```

Every byte is XORed. The result is ORed into `diff`. The loop always runs to completion. The comparison at the end is a single branch on a non-secret value.

## The length problem

The length check at the top is itself a leak — it reveals whether the secret has the expected length. In some contexts, this matters. In others, it doesn't.

For most use cases, the length of a secret is not sensitive (a 32-byte key is a 32-byte key). The contents are. Constant-time comparison of contents while leaking length is the standard tradeoff.

If length must also be protected, you're in padding territory — compare against a fixed-size buffer padded with random bytes. This adds complexity and is rarely necessary.

## Compiler interference

The compiler might optimize the loop back into a short-circuit if it can prove the early exit produces the same result. This is why `subtle` and similar crates use inline assembly or volatile reads to prevent the compiler from undoing the constant-time property.

```rust
fn ct_eq_volatile(a: &[u8], b: &[u8]) -> bool {
    if a.len() != b.len() {
        return false;
    }
    let mut diff: u8 = 0;
    for (x, y) in a.iter().zip(b.iter()) {
        let xored = x ^ y;
        unsafe { core::ptr::read_volatile(&xored) };
        diff |= xored;
    }
    diff == 0
}
```

The volatile read forces the compiler to materialize each XOR result, preventing it from collapsing the loop.

## What memguard-rs provides

The crate exposes a `ct_eq` function built on volatile reads with a compiler fence, wrapped in a safe API. Users don't need to think about compiler optimizations or inline assembly. The comparison is constant-time by construction, and the type system prevents accidental exposure of the underlying bytes.
