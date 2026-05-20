# Ownership and Lifetimes
Rust prevents data races and dangling borrows

Example: ```
let mut data = vec![1, 2, 3];

let first = &data[0];

data.push(4);

println!("{data}");```

`data.push(4)` causes thte storage of data to be reallocated and first is now a dangling pointer and also prevents a potential data race if `data.push(4)` fails

This is why Rust requires references to freeze the referent and its owners.

## References
- Shared reference: `&`
- Mutable reference: `&mut`

A reference cannot outlive its referent and a mutable reference cannot be aliased.

## Aliasing
Variables and pointers alias if they refer to overlapping regions of memory. Alias analysis is important because it lets the compiler perform useful optimizations such as:
- keeping values in registers by proving no pointers access the values memory
- eliminating reads by proving some memory hasn't been written to since last read
- eliminating writes by proving some memory is never read before the next write to it
- moving or reordering reads and writes by proving they don't depend on each other

These optimizations also tend to prove the soundness of bigger optimizations such as loop vectorization, constant propagation, and dead code elimination. Writes are the primary hazards for optimizations, only prevention from moving read to any other part of the program is possibilty of re-ordering with a write to same location. Value of local variable can't be aliased by things that existed before it was declared. A full aliasing model for Rust must also take into consideration things like function calls (may perform implicit mutations), raw pointers (no aliasing requirements on their own), and UnsafeCell (lets referent of an `&` be mutated)

## Lifetimes
Rust enforces these rules through lifetimes which are named regions of code that a reference must be valid for. There may even be holes in these paths of execution, as it's possible to invalidate a reference as long as it's reinitialized before it's used again. Types which contain references may also be tagged with lifetimes so that Rust can prevent them from being invalidated as well. In most examples, the lifetimes coincide with scopes but there are cases in which this is false.
