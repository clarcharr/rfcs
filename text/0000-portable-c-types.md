- Feature Name: portable_c_types
- Start Date: (fill me in with today's date, YYYY-MM-DD)
- RFC PR: (leave this empty)
- Rust Issue: (leave this empty)

# Summary
[summary]: #summary

This RFC extends RFC 2521 (`core::ffi::c_void`) by creating opaque, `repr(transparent)` structs in `core::ffi` for the other `c_*` types, like `c_int`.

# Motivation
[motivation]: #motivation

Right now, 

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

Rust's primitive types define exactly what size they are, to avoid all ambiguity. For example, while C defines `char`, `short`, `int`, and `long`, these are only sometimes 8, 16, 32, and 64 bits, respectively. In languages like Java, which borrow from C, these types do correspond to these sizes, despite the fact that C does not. In Rust, we use `i8`, `i16`, `i32`, and `i64` to describe these types.

Conversion between Rust's native integer types and FFI types can lead to subtle portability bugs if you're not careful. To prevent this, Rust exports opaque versions of these types in `std::ffi` which allow safe interop with the C types. Conversions to and from these types are done using the `From` and `TryFrom` traits.

For example, a `main` function in Rust might look like:

```
use std::ffi::{c_char, c_int};
use std::slice;

#[no_mangle]
extern fn main(argc: c_int, argv: *const *const c_char) -> c_int {
    let arg_count = usize::try_from(argc).expect("number of arguments should fit in usize");
    let args = unsafe { slice::from_raw_parts(argv, arg_count) };
    for arg in args {
        let arg = unsafe { CStr::from_ptr(arg).to_string_lossy() };
        println!("Got an argument: {}", arg);
    }
    0
}
```

Note that in this case, we have to do `usize::try_from(argc)`. Even though we can rest assured that on most systems, `argc` won't be negative and will fit in `usize`, the types themselves tell us that we have to check anyway.

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

The `core::ffi` module will be augmented with the following types, which will map exactly to their C equivalents:

* `c_char` => `char`
* `c_schar` => `signed char`
* `c_uchar` => `unsigned char`
* `c_short` => `short`
* `c_ushort` => `unsigned short`
* `c_int` => `int`
* `c_uint` => `unsigned int`
* `c_long` => `long`
* `c_ulong` => `unsigned long`
* `c_longlong` => `long long`
* `c_ulonglong` => `unsigned long long`
* `c_float` => `float`
* `c_double` => `double`
* `c_longdouble` => `long double`

All of these (with the exception of `c_longdouble`) will be defined using a `repr(transparent)` struct. Conversions to and from Rust types will be done using the `From` and `TryFrom` traits, strictly adhering to the C definitions. This means that, for example, `From<u32> for c_uint` will not exist, but `From<u16> for c_uint` will.

Right now, `c_longdouble` is the one exception, as a native `f80` type or `f128` would be required to represent this on some platforms. Instead, until such a type exists, `c_longdouble` will be implemented using a lang item and only allow lossy conversion to `f32` and `f64`, with exact conversion from `f32` and `f64`.

In addition to `From` conversions, all of these types will be `Copy`, and all types except `c_longdouble` will derive `Ord`, `Hash`, `Debug`, and `Display`, inheriting the versions for their primitive counterparts.

# Drawbacks
[drawbacks]: #drawbacks

Why should we *not* do this?

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

- Why is this design the best in the space of possible designs?
- What other designs have been considered and what is the rationale for not choosing them?
- What is the impact of not doing this?

# Prior art
[prior-art]: #prior-art

Discuss prior art, both the good and the bad, in relation to this proposal.
A few examples of what this can include are:

- For language, library, cargo, tools, and compiler proposals: Does this feature exist in other programming languages and what experience have their community had?
- For community proposals: Is this done by some other community and what were their experiences with it?
- For other teams: What lessons can we learn from what other communities have done here?
- Papers: Are there any published papers or great posts that discuss this? If you have some relevant papers to refer to, this can serve as a more detailed theoretical background.

This section is intended to encourage you as an author to think about the lessons from other languages, provide readers of your RFC with a fuller picture.
If there is no prior art, that is fine - your ideas are interesting to us whether they are brand new or if it is an adaptation from other languages.

Note that while precedent set by other languages is some motivation, it does not on its own motivate an RFC.
Please also take into consideration that rust sometimes intentionally diverges from common language features.

# Unresolved questions
[unresolved-questions]: #unresolved-questions

* Should `c_longdouble` be left to a future RFC, or added as a lang item?
* Should `c_longdouble` implement `Ord`, `Hash`, `Debug`, and `Display` before its underlying types are usable in stable Rust?
* Should other C types, like `size_t` and `uintptr_t` (which aren't guaranteed to be equal to `usize`), be included as well?
* Where do `as` casts fit into this? Should this be blocked on truncating/rounding conversion traits that replace `as`, or one which allows `as` for arbitrary types?
