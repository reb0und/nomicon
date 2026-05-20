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

Aliasing a value is creating another value that points to the same data.

## Lifetimes
Lifetimes are generic type annotations on borrows denoting the lifetime of a borrow relative to its data and other borrows, enabling lifetime comparison and preventing using anohter borrow in place of another in result. It also prevents producing a borrow that has a shorter or different lifetime than specified. 

Rust enforces aliasing rules denoting there can exist many immutable borrows but only a single mutable borrow at once. Lifetimes are named regions of code that a reference must be valid for and correspond to paths of execution in the program. There may be holes in these paths of execution, as it's possible to invalidate a reference as long as it's reinitialized before reuse. Types containing references must also be tagged with lifetimes so Rust can prevent them from being invalidated. More complex cases of lifetimes are those that don't coincide with scope.

Do not need to explicitly name lifetimes in function body because they exist for local context unless dropped early. When crossing function boundary, need to begin talking about lifetimes. One piece of sugar is `let` statements implicitly introduce a scope. This only matters when variables refer to each other.

Example: ```
let x = 1;
let y = &x;
let z = &y;

'a: {
    let x = 1_u8;

    'b: {
        let y: &'b u8 = &x;

        'c: {
            let z: &'c &'b u8 = &y;
        }
    }
}```

Passing references to outer scopes will cause Rust to infer larger lifetimes:
Example: ```
let x = 0;
let z;
let y = &x;
z = y;

'a: {
    let x = 1_u8;

    'b: {
        let z: &'b u8;

        'c: {
            // Since &x: 'b
            let y: &'b = &x;
            z = y;
        }
    }
}```

### Example: references that outlive referents
Example: ```
fn as_str(data: &u8) -> &str {
    let s = format!("{}", data);

    &s
}```

This produces a borrow to a local heap allocation in `format!` which is a dangling borrow. This expands to the following:
Example: ```
fn as_str<'a>(data: &'a u8) -> &'a str {
    'b: {
        let s = format!("{}", data);

        &'a s
    }
}```

This is a dangling borrow becuase the data's lifetime (lifetime outliving function is `'a` while the produced data within `'b`. The resulting type signature of `&'a str` implies `&'a str` is producible from `&'a u8` without an allocation and exists within the scope of the reference `data` originated in, which is untrue. This then computes `s` and produces a borrow from it but this borrow does not outlive the function.

The correct way to format this function is to perform an allocation and produce an owned value.

### Example: aliasing a mutable reference
Example: ```
let mut data = vec![1, 2, 3];

// Immutable borrow
let x = &data[0];

// Mutable borrow
data.push(4);
println!("{}, x");

'a: {
    let mut data = vec![1, 2, 3];

    'b: {
        // 'b is big enough for borrow
        let x: &'b u8 = Index::index::<'b>(&'b data, 0);

        'c: {
            // 'c is big enough for borro because temporary scope
            Vec::push(&'c mut data, 4);
        }

        println!("{}", x);
    }
}```

This violates aliasing rules by creating a mutable borrow with an immutable borrow. This is also disallowed because a mutable borrow is created under a scope within `'c` while there is an immutable borrow in `'b`. This is rejected because the `'b` data must still be alive and this is not confirmed by using `'c` because it is a different lifetime.

### The area covered by a lifetime
A borrow is alive from its creation to its last use. The borrowed value must outlive borrows that are alive.

Example: ```
let mut data = vec![1, 2, 3];

let x = &data[0];

// x is discarded because the borrow is last used here
println!("{}", x);

// create and use immutable borrow
data.push(4);```

If value has destructor, the destructor is run at end of scope and running destructor is considered a use.
Example: ```
#[derive(Debug)]
struct X<'a>(&'a i32);

impl Drop for X<'_> {
    fn drop(&mut self) {}
}

let mut data = vec![1, 2, 3];

// create immutable borrow
let x = X(&data[0]);

println!("{}", x);

// create mutable borrow, violating aliasing rules
data.push(4);

// implicitly drop X, immutable borrow is still alive```

A way to indicate `x` is no longer valid and prematurely drop `x` before end of scope is using `drop(x)` before mutable borrow is created in `data.push(4)`

Example: ```
let mut data = vec![1, 2, 3];
let x = &data[0];

if some_condition {
    // last use of x, valid
    println!("{}", x);
    data.push(4);
} else {
    // no x usage
    data.push(4);
}```

## Limits of Lifetimes
Example: ```
#[derive(Debug)]
struct Foo;

impl Foo {
    fn mutate_and_share(&mut self) -> &Self { &*self }
    fn share(&self) {}
}

fn main() {
    let mut foo = Foo;
    let loan = foo.mutate_and_share();
    foo.share();

    println!("{:?}", loan);
}```

Example: ```
#[derive(Debug)]
struct Foo;

impl Foo {
    fn mutate_and_share<'a>(&'a mut self) -> &'a Self { &'a *self }
    fn share<'a>(&'a self) {}
}

fn main() {
    'b: {
        let mut foo = Foo;

        'c: {
            let loan: &'c Foo = Foo::mutate_and_share::<'c>(&'c mut foo);

            'd: {
                // &'c mut Foo is still alive at this point because 'c: 'd and loan is used afterwards, violating aliasing rules
                Foo::share::<'d>(&'d foo);
            }
        }
    }

    println!("{:?}", loan);
}```

The lifetime system extends `&mut foo` to have lifetime `c` because of `loan` and `mutate_and_share`'s signature.

### Improperly reduced borrows
This fails to compile because `map` is borrowed twice and cannot infer the first borrow ceases to be needed before the second one. This is caused by Rust conservatively falling back to using a whole scope for the first borrow.

Example: ```
fn get_default<'m, K, V>(map: &'m mut HashMap<K, V>, key: K) -> &'m mut V
where
    K: Clone + Eq + Hash,
    V: Default,
{
    match map.get_mut() {
        Some(value) => Value,
        None => {
            map.insert(key.clone(), V::default());
            map.get_mut(&key).unwrap();
        },
    }
}```

Because of lifetime restrictions, the first `map.get_mut()`'s lifetime overlaps other mutable borrows, resulting in a compile error. 

## Lifetime Elision
Rust allows lifetime elision in function signatures.

```
&'a T
&'a mut T
T<'a>```

Lifetime positions can appear as either input or output
- for `fn` declarations, `fn` types, and `Fn`, `FnMut`, `FnOnce` traits, input refers to types of formal arguments, while output refers to result types. `fn foo(s: &str) -> (&str, &str)`, has elided one lifetime in input position to two lifetimes in output position. Input positions of a `fn` method definition do not include lifetimes from method's `impl` header (nor lifetimes in trait header)
- For `impl` headers, all types are input. `impl Trait<&T> for Struct<&T>` has elided two lifetimes in input position, `impl Struct<&T>` has elided one

Elision rules:
- Each elided lifetime in input position becomes a distinct lifetime parameter
- If there is exactly one input lifetime position (elided or not), that lifetime is assigned to all elided output lifetimes
- If there are multiple input lifetime positions, but one of them is `&self` or `&mut self`, the lifetime of `self` is assigned to all elided output lifetimes.
- Otherwise it is an error to elide an output lifetime

## Unbounded Lifetimes
Unsafe can often produce references from nothing and are considered unbounded. The most common source of this is taking a reference to a dereferenced raw pointer, which produces a reference with an unbounded lifetime. This lifetime becomes as big as context demands, this is more powerful than becoming `'static` but the unbound lifetime will take `&'a &'a T` as needed. For most purposes, unboundeded lifetimes can take `'static`

No reference is `'static`, which is likely wrong. Unbounded lifetimes should be bounded as quickly as possible, especially across function boundaries. Given a function, any output lifetimes that don't derive from inputs are unbounded.

Example: ```
fn get_str<'a>(s: *const String) -> &'a str {
    unsafe { &*s }
}

fn main() {
    let soon_dropped = String::from("xyz");
    let dangling = get_str(&soon_dropped);
    drop(soon_dropped);
    println!("invalid str: {}", dangling);
}```

Easiest way to avoid unbounded lifetimes is to use lifetime elision at the function boundary. If an output lifetime is elided, it must be bounded by an input lifetime. Within a function, bounding lifetimes is more error-prone. Safest and easiest way to bound a lifetime is to return it from a function with a bound lifetime. Alternatively, reference can be placed in location with a specific lifetime.

## Higher Ranked Trait Bounds (HRTBs)
Example: ```
struct Closure<F> {
    data: (u8, u16),
    func: F,
}

impl<F> Closure<F>
    where for<'a> F: Fn(&'a (u8, u16)) -> &'a u8,
{
    fn call(&self) -> &u8 {
        (self.func)(&self.data)
    }
}

fn do_it (data: & (u8, u16)) -> & u8 {
    &data.0
}

fn main() {
    let clo = Closure { data: (0, 1), func: do_it };
    println!("{}", clo.call());
}```

HRTBs express lifetimes in trait bounds

Example: `where for<'a> F: Fn(&'a (u8, u16)) -> &'a u8`
Alternatively:
`where F: for<'a> Fn(&'a (u8, u16)) -> &'a u8`

`for<'a>` is read as "for all choices of `'a`" and produces an infinite list of trait bounds that F must satisfy. There aren't many places outside of `Fn` where HRTBs are encountered.
