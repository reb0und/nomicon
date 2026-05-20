# Data Represenation
## repr(Rust)
All types have alignment specified in bytes, indicating addresses valid to store value at. Alignment of `n` must only store value at address multiple of `n`. Primitives typically aligned to their size except for x86, `u64` and `f64` are 4-byte aligned

Type size must always be multiple of alignment. This ensures that array of a type may always be indexed by offsetting by a multiple of its size. Size and type alignment may not be statically known in case of DSTs.

### Rust Composite Types:
- structs (named product types)
- tuples (anonymous product types)
- arrays (homogeneous product types)
- enums (named sum type)
- unions (untagged unions)

By default, composite structures have an alignment equal to the maximum of their fields' alignments. Rust will consequently insert padding where necessary to ensure all fields are properly alignmed and that overall type's size is a multiple of its alignment. Arrays are densely packed and in-order. Structs of same type are guarnateed to have identical layouts however same composition by different type may be different.

- Rust monomorphizations reduce field space by reordering fields.
- Enums are similar but include a tag indicating variant
- This may be inefficient when there is a null pointer optimization for an enum consisting of a single outer unit variant and another non-nullable pointer variant. A null pointer can be interpreted as the unit variant, making the allocation size the same as the other variant
- Nested enums can be grouped under a single tag as another optimization

## Exotically Sized Types
Types with size unknown at compile time

### Dynamically Sized Types (DSTs)
Rust supports DSTs which are dynamically sized types without a statically known size or alignment. DSTs must only exist behind a pointer.

DSTs:
- trait objects
- slices: `[T]. str`, etc

Trait objects represent some type that implements the trait it specifies. Original type is erased in favor of runtime reflection with a vtable containing all necessary type information. The information completing trait object pointer is vtable pointer. Pointee runtime size can be requested from vtable.

Slice is contiguous view of storage (typically array or `Vec`). Information completing slice pointer is number of elements or length. Runtime size of pointee is statically known size of element mulitplied by number of elements.

### Zero Sized Types (ZSTs)
Rust allows types to be specified that occupy no space. An operation that produces or stores a ZST can reduce to a no-op.

Example: Given a `Map<Key, Value>`, can implement `Set<Key>` using `Map<Key, ()>`

Pointer offsets are no-ops

### Empty Types
Rust also enables types that cannot be instantiated, can only exist at type level, not value. Empty types can be declared by specifying an enum with no variants.

Example: `enum Empty {}`

Primary motivation for empty types is type level unreachability, for example, returning a result where specific case is infallible. Can communicate this at type level with `Result<T, Empty>`. Result is unwrappable without risking panics or yielding an `Err`. This is optimized to `T` since `Err` case does not exist, this optimization is not guaranteed. Raw pointers to empty types are valid to construct but dereferencing is UB.

## Alternative representations
### repr(C)
Does what C does. Matches order, size, alignment of fields from C and C++. Type is also passed across `extern "C"` function call boundaries same way C would pass corresponding type. Any type passed through FFI boundaries should have `repr(C)`. This is also necessary for type reinterpretation. `repr(C)` can be applied to types that will be nonsensical or problematic if passed through the FFI boundary.
- ZSTs are still zero-sized
- DST pointers (wide pointers) and tuples are not concepts in C and are never FFI-safe 
- Enums with fields also aren't concept in in C but valid type bridging is defined
- If `T` is an FFI-safe non-nullable pointer type, `Option<T>` is guaranteed to have same layout and ABI as `T` and is therefore FFI-safe.
- Tuple structs are like structs but fields are unnamed
- `repr(C)` is equivalent to one of `repr(u*)` for fieldless enums. Chosen sign and size is default enum size and sign for target platform's C ABI. Enum represenation in C is implementation defined, this varies.
- Fieldless enums with `repr(C)` or `repr(u*)` still may not be set to an integer without a corresponding variant

### repr(transparent)
`#[repr(transparent)]` can only be used on a struct or a single-variant enum that has a single non-zero-sized field (there may be additional zero-sized fields). The effect is that the layout and ABI of the whole struct or enum is guaranteed to be the same as that one field.

There is a `transparent_unions` nightly feature to apply transparency to unions

The goal is to enable transmutation between the single field and the struct or enum. An example is the `UnsafeCell`, which can be transmuted into the type it wraps, however, its ABI is not guaranteed to match. Also, passing the struct/enum wrapped value through FFI yields the inner value

### repr(u*), repr(i*)
These specify the size and sign to make a fieldless enum. If the discriminant overflows the integer it has to fit in, it will produce compile-time error. Can manually ask Rust to allow this by setting the overflowing element to explicitly be 0. Rust will not allow creating enum where two variants have same discriminant.

Fieldless enum means enum does not have data in any of its variants. A fieldless enum without a `repr` is still a Rust native type and lacks a stable layout or represenation. Adding a `repr(u*)`/`repr(i*)` causes similar treatment to specified integer type for layout purposes except compiler will use knowledge of invalid values to optimize enum layout, such as when enum is wrapped in `Option<T>`. The function call ABI for these types is unspecified, except that across `extern "C"` calls they are ABI-compatible with C enums of the same sign and size.

If the enum has fields, the effect is similar to `repr(C)` in that there is a defined layout of the type, making it possible to pass the enum to C code or access the type's raw representation and directly manipulate its tag and fields. These `repr`s have no effect on a struct.

Adding `repr(u*)`, `repr(i*)`, or `repr(C)` to an enum with fields suppresses the null-pointer optimization 

### repr(packed), repr(packed(n))
`repr(packed(n))` (where `n` is a power of 2) forces the type to have an alignment of at most `n`. `repr(packed)` is equivalent to `repr(packed(1))` which removes padding and only align the type to a byte.

### repr(align(n))
`repr(align(n))` (where `n` is a power of two) forces the type to have an alignment of at least `n`, enabling things such as ensuring neighboring elements of an array never share the same cache line with each other. This is a modifier on `repr(C)` and `repr(Rust)` and is incompatible with `repr(packed)`
