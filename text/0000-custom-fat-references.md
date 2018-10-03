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

This RFCS allows users to define structs as follows:

```rust
struct &MyFatRef {
    /* body */
}
```

This new type is called a "reference struct". This has notably different properties than a normal
struct:
- Any types within the struct must implement `Copy`.
- The type's structure entirely replaces the structure of references to that type.
    TODO PHRASE THIS BETTER
    ```rust
    struct &BigRef([u128; 100]);
    assert_eq!(mem::size_of::<&BigType>(), mem::size_of::<[u128; 100]>());

    struct &SmolRef;
    assert_eq!(mem::size_of::<&SmolRef>(), 0);
    ```
- Any instances of a reference struct *must* be defined behind a reference.
    ```rust
    struct &MyFatRef(u64);
    let _ = &MyFatRef(36); // Allowed
    let _ = MyFatRef(63); // Not allowed
    ```
- Except as an intermediate step, dereferencing a reference struct to obtain the underlying type
  is illegal.

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

## Case study: implementing a 2D slice

```rust
struct Array2D {
    data: Vec<u8>,
    width: usize
}

struct &Slice2D {
    start: *mut u8,
    width: usize,
    // The distance between the start of one segment and the start of the next one.
    stride: usize,
    height: usize
}

impl Array2D {
    fn new(width: usize, height: usize) -> Array2D {
        Array2D {
            data: vec![0; width * height]
        }
    }

    fn width(&self) -> {self.width}
    fn height(&self) -> {self.data.len() / self.width}
}

impl Deref for Array2D {
    type Target = Slice2D;
    fn deref(&self) -> &Slice2D {
        &Slice2D {
            start: self.data.as_ptr(),
            width: self.width(),
            stride: self.width(),
            height: self.height()
        }
    }
}

impl Slice2D {
    fn get_columns(&self, column_range: Range<usize>) -> Option<&Slice2D> {
        if column_range.start > column_range.end {
            None
        } else if column_range.end > self.width {
            None
        } else {
            Some(&Slice2D {
                start: unsafe{ self.start.offset(column_range.start) },
                width: column_range.len(),
                stride: self.stride,
                height: self.height
            })
        }
    }

    fn get_rows(&self, row_range: Range<usize>) -> Option<&Slice2D> {
        if row_range.start > row_range.end {
            None
        } else if row_range.end > self.height {
            None
        } else {
            Some(&Slice2D {
                start: unsafe{ self.start.offset(row_range.start * self.stride) },
                width: self.width,
                stride: self.stride,
                height: row_range.len()
            })
        }
    }
}

impl Index<usize> for Slice2D {
    type Output = [u8];
    fn index(&self, index: usize) -> &[u8] {
        let slice_2d = self.get_rows(index..index + 1).unwrap();
        assert_eq!(1, slice_2d.height);
        unsafe{ slice::from_raw_parts(slice_2d.start, slice_2d.width) }
    }
}
```

Explain the proposal as if it was already included in the language and you were teaching it to another Rust programmer. That generally means:

- Introducing new named concepts.
- Explaining the feature largely in terms of examples.
- Explaining how Rust programmers should *think* about the feature, and how it should impact the way they use Rust. It should explain the impact as concretely as possible.
- If applicable, provide sample error messages, deprecation warnings, or migration guidance.
- If applicable, describe the differences between teaching this to existing Rust programmers and new Rust programmers.

For implementation-oriented RFCs (e.g. for compiler internals), this section should focus on how compiler contributors should think about the change, and give examples of its concrete impact. For policy RFCs, this section should provide an example-driven introduction to the policy, and explain its impact in concrete terms.

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

## `impl`s

Like `[T]`, implementations on `&Types` can be created without the leading `&`:

```rust
struct &MyFatRef { }

impl MyFatRef {
    fn foo(&self) {
        println!("called immutably!");
    }

    fn foo_ut(&mut self) {
        println!("called mutably!");
    }
}
```

Functions defined inside cannot take `self` as a parameter; they must use `&self` or `&mut self`.

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
