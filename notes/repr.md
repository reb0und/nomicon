# Data Layout

## repr(Rust)
All types have alignent specified in bytes, alignment of a type specifies what address are valid to store a type at. Primitives are usually aligned to their size. A type's size must always be a multiple of it's alignment, ensuring that an array of the type may always be indexed by offsetting a multiple of it's size. Note that the size and alignment of a type may not be known statically in the case of DSTs.

### Composite Types
- Struct
- Enum
- Tuple
- Array
- Union

- By default, composite structures have an alignment equal to the maximum of their fields' alignments
- Rust will insert padding where necessary to ensure proper alignment among fields
- There is no indirection for types in structs, all data is stored within the struct
- Data layout is not specified by default
- Rust guarantees that two instances of the same struct have their data laid out in the same way but does not guarantee same field ordering as a struct of same composition
- Optimal use of space requires monomorphizations to have different field orderings
- Enum representation can be laid out as a struct including a numeric tag
    - This representation is not always efficient, for example, Option<&T> can be represented at &T where Option::None == nullptr

## Exotically Sized Types
Most of types are expected to have a statically known and positive size but this isn't always the case in Rust

### Dynamically Sized Types (DSTs)
- Rust supports DSTs, which are types without a statically known size or alignment, these types can only exist behind a pointer since these attributes are unknown
- Any pointer to a DST consequently becomes a wide pointer consisting of the pointer and the information that completes them

Two major DSTs in Rust:
- trait objects: `dyn Foo`
- slices: `[T]`, `str`, etc

- With trait objects, the exact original type is erased in favor of runtime reflection with a vtable containing all the information necessary to use the type
- The information that completes a trait object is the vtable pointer, the runtime size of the pointee can be dynamically requested from the vtable
- A slice contains a view into some contiguous storage, typically an array or `Vec`, the information completing a slice pointer is the number of elements it points to
- The runtime size of the pointee is the statically known size of an element multiplied by the number of elements
- DST fields make a composite type a DST
- Only supported way to properly create a DST is by making the type generic and performing an unsizing coercion

### Zero Sized Types (ZSTs)
Rust also allows types to be specified that occupy no space
- Example: `struct Nothing;`
- Example: `struct LotsOfNoting {
    foo: Nothing,
    qux: (),
    baz: [u8; 0],
}`
- Any operation that produces a ZST can be reduced to a no-op since it occupies no space
- Example of use is when attempting to create a set from a map
    - Given a `Map<Key, Value>` and trying to form a `Set<Key>` from `Map<Key, Useless>`, this would typically require allocating space for `Useless` but can say that `Set<Key>` = `Map<Key, ()>`, now Rust statically knows that every load and store is useless and has no allocation
- ZST sizing is unimportant in safe code but in unsafe, must worry about consequences of types without size, in particular pointer offsets are no-ops and allocators typically require a non-zero size
- ZSTs must be non-null and suitably aligned

### Empty Types
Rust also enables types that cannot be instantiated and can only be used at the type level but never the value level
- Empty types can be declared by specifying an enum with no variants
    - Example: `enum Void {}`
- This can be used to represent a case that is infallible, useful for type-level unreachability
    - Example: returning a Result but a specific case is infallible, can return `Result<T, Void>`
    - API users can unwrap knowing that it is statically impossible for this value to be an `Err` as this would require providing a value of `Void`
    - `Result<T, Void>` can be represented as  `T` because the `Err` case doesn't exist
- Raw pointers to empty types are considered valid to construct, but dereferencing them is UB

### Alternative representations
#### repr(C)
Matches order, size, and alignment of fields from C, the type is also passed along `extern "C"` function call boundaries the same way C would pass the corresponding type
- Any type passed through an FFI boundary should have `repr(C)`
- This also enables reinterpreting data as a different type
- `repr(C)` can be applied to types that will be nonsensical or problematic if passed through the FFI boundary such as exotic types
    - ZSTs are still zero-sized
    - DST pointers (wide pointers) and tuples are not a concpet in C and are therefore never FFI-safe
    - Enums and fields have a valid bridging of types
    - If `T` is an FFI-safe non-nullable pointer type, `Option<T>` is guaranteed to have the same layout and ABI as `T` and is therefore FFI-safe
        - This covers `&`, `&mut`, and function pointers, all of which can never be null
    - Tuple structs are like structs with regard to `repr(C)`, the only difference from a struct is that the fields aren't named
    - `repr(C)` is equivalent to one of `repr(u*)` for fieldless enums. The chosen size and sign is the default enum size and sign for the target platform's C ABI
        - The enum representation in C is implementation defined which makes this a guess, this may be incorrect under certain flags
    - Fieldless enums with `repr(C)` or `repr(u*)` may not be set to an integer value without a corresponding variant
        - It is UB to construct an instance of an enum that does not match one of its variants, allowing exhaustive matches to be written and compiled as normal 

#### repr(transparent)
