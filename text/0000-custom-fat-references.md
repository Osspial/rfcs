- Feature Name: custom_fat_references
- Start Date: (fill me in with today's date, YYYY-MM-DD)
- RFC PR: (leave this empty)
- Rust Issue: (leave this empty)

# Summary
[summary]: #summary

One paragraph explanation of the feature.

# Motivation
[motivation]: #motivation

Why are we doing this? What use cases does it support? What is the expected outcome?

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

Explain the proposal as if it was already included in the language and you were teaching it to another Rust programmer. That generally means:

- Introducing new named concepts.
- Explaining the feature largely in terms of examples.
- Explaining how Rust programmers should *think* about the feature, and how it should impact the way they use Rust. It should explain the impact as concretely as possible.
- If applicable, provide sample error messages, deprecation warnings, or migration guidance.
- If applicable, describe the differences between teaching this to existing Rust programmers and new Rust programmers.

For implementation-oriented RFCs (e.g. for compiler internals), this section should focus on how compiler contributors should think about the change, and give examples of its concrete impact. For policy RFCs, this section should provide an example-driven introduction to the policy, and explain its impact in concrete terms.

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

Allow defining structs, unions, and enums as follows:

```rust
struct &MyFatRef {
    /* body */
}
```

When a type is defined with the above pattern, the following semantics apply to it:
- The type's structure entirely replaces the structure of references to that type.
    TODO PHRASE THIS BETTER
    ```rust
    struct &BigRef([u128; 100]);
    assert_eq!(mem::size_of::<&BigType>(), mem::size_of::<[u128; 100]>());

    struct &SmolRef;
    assert_eq!(mem::size_of::<&SmolRef>(), 0);
    ```
- Any instances of such a type *must* be defined behind a reference.
    ```rust
    struct &MyFatRef(u64);
    let _ = &MyFatRef(36); // Allowed
    let _ = MyFatRef(63); // Not allowed
    ```

Note that, for a given `struct NormalStruct(u64)`, taking a `&NormalStruct(_)` and a `&MyFatRef(_)`
are fundamentally different operations, as `&MyFatRef(_)` has ownership of the data in the
`&MyFatRef` struct, while `&NormalStruct(_)` does not:

```rust
// This results in a compile error, as you're attempting to borrow a value that goes
// out of scope when the function returns.
fn return_ref_bad(i: u64) -> &'static NormalStruct {
    &NormalStruct(i)
}

// However, this DOES compile successfully. This is because the `u64` is stored INSIDE
// the `&MyFatRef` struct.
fn return_ref_good(i: u64) -> &'static MyFatRef {
    &MyFatRef(i)
}

// Ownership-wise, `return_ref_good` is identical to the below function, which constructs
// a `&'static [T]` from an arbitrary integer. The integer gets stored inside the `&[T]`,
// not in any pointed-to `[T]`.
fn return_ref_slice<T(i: isize) -> &'static [T] {
    unsafe{ slice::from_raw_parts(i as *const T, 0) }
}
```


This RFC also defines a new lang item: `ops::FatRef`:

```rust
#[lang = "fat_ref"]
unsafe trait FatRef {
    type Target: ?Sized;

    fn points_to(&self) -> *mut Self::Target;
}
```

TODO: WHAT SHOULD BE DONE ABOUT USERS IMPLEMENTING `RefType` FOR NON-`&Type` TYPES?

`Deref` and `DerefMut` are automatically implemented for `T: FatRef`, and said implementations are
explicitly non-specializable.

## Why not have users implement `Deref`/`DerefMut` directly?

Currently, references expose semantics that aren't supported with `DerefMut`: using mutable references,
it's possible to *borrow mutable data from an immutable variable*:

```rust
// We have a mutable variable `number`.
let mut number = 43.0;

// We now take a mutable reference to `number`. The `ref_mut` variable ITSELF
// is not mutable, but it still allows us to mutate the underlying data.
let ref_mut = &mut number;
*ref_mut -= 2.0;
```

In contrast, take a given smart pointer struct, implementable today:

```rust
struct OurSmartPtr<'a>(&'a mut f32);

impl<'a> Deref for OurSmartPtr<'a> {
    type Target = f32;
    fn deref(&self) -> &f32 {
        self.0
    }
}
impl<'a> DerefMut for OurSmartPtr<'a> {
    fn deref_mut(&mut self) -> &mut f32 {
        self.0
    }
}
```

If we tried to implement the original code with an immutable `OurSmartPtr` variable:

```rust
let smart_ptr = OurSmartPtr(&mut number);
*smart_ptr += 0.9;
```

The compiler throws an error:

```rust
error[E0596]: cannot borrow immutable local variable `smart_ptr` as mutable
  --> src/main.rs:20:6
  |
1 |     let smart_ptr = OurSmartPtr(&mut number);
  |         --------- consider changing this to `mut smart_ptr`
2 |     *smart_ptr += 0.9;
  |      ^^^^^^^^^ cannot borrow mutably
```

This is because `deref_mut()` takes a mutable reference to `self`, with which it *is* possible to
mutate the smart pointer struct. Forcing this behavior would cause all custom fat references to
behave fundamentally differently to in-language fat references.

## Why does `points_to` return a pointer?

Rust currently doesn't expose any mechanism for defining a reference to be *either* mutable or
immutable, or allowing references being generic over mutability.

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

- What parts of the design do you expect to resolve through the RFC process before this gets merged?
- What parts of the design do you expect to resolve through the implementation of this feature before stabilization?
- What related issues do you consider out of scope for this RFC that could be addressed in the future independently of the solution that comes out of this RFC?
