# Unsafe

## How Safe and Unsafe Interact
- Safe Rust calls unsafe under specific constraints that prevent UB
- Any unsafe code that uses safe code must defend against incorrect behavior

## What Unsafe Can Do
- Dereference raw pointers
- Call unsafe functions (such as C functions, compiler intrinsics, and the raw allocator)
- Access or mutate a mutable statics
- Implement an unsafe trait
- Access fields of a union

Rust considers it safe to do the following:
- Deadlock
- Have a race condition
- Leak memory
- Overflow integers (with the built-in operators such as `+`, etc)
- Abort the program

## Working with Unsafe
- Only way to limit scope of unsafe code is at module boundary with privacy
    - Only module defining unsafe code calls it
- Unsafe code must trust Safe code but should not trust generic Safe code
- Privacy prevents from having to trust all safe code from interrupting trusted state
- Unsafe Rust cannot deal with Safe Rust without care as it may cause UB
