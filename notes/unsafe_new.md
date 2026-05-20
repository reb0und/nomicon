# Unsafe New

## How Unsafe and Safe Interact
- Unsafe declares existence of unchecked contracts on functions and traits
- Programmer must uphold safety
- Unsafe marks programmer is responsible for safety of item
- Safe interfaces built on top of implementations must be rigorously manually checked for proper conditions to avoid invoking UB
- Separation because of soundness property of Safe Rust: Safe Rust cannot cause UB
- Safe Rust must trust all Unsafe abstractions while Unsafe cannot trust Safe Rust without care
- If Unsafe cannot defend against Safe, implementation must be Unsafe 

## What Unsafe Rust Can Do
### UB
- Dereference raw pointers
- Call `unsafe` functions
- Implement `unsafe` traits
- Access or modify mutable statics
- Access fields of `union`s

### Safe UB
- Deadlock
- Race condition
- Leak memory
- Overflow integers
- Abort program

## Working with Unsafe
- Can limit mutating variants through module privacy, prevent modifying internals
