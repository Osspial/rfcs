- Feature Name: lang_level_licenses
- Start Date: (fill me in with today's date, YYYY-MM-DD)
- RFC PR: [rust-lang/rfcs#0000](https://github.com/rust-lang/rfcs/pull/0000)
- Rust Issue: [rust-lang/rust#0000](https://github.com/rust-lang/rust/issues/0000)

# Summary
[summary]: #summary

- Add a new compiler built-in macro:
    + `licenses!()`: a list of all in-project and upstream licenses.
- Add to the `rustc` CLI:
    + `rustc --emit licenses`: emit the human-readable license list as part of the compiler output
    + `rustc --emit licenses-json`: emit the JSON-formatted license list as part of the compiler output
    + `rustc --licenses PATH`: path to a JSON-formatted list of the licenses used by user-compiled code
- Add to `cargo`
    + `[licenses]` section in `Cargo.toml`: configure how Cargo generates the license list passed to `rustc`

# Motivation
[motivation]: #motivation

Every single piece of code in the Rust ecosystem, from a simple "Hello, World" example to a complex application with hundreds of dependencies through Cargo, depends on licensed open-source software. At the lowest level, projects depend on `libcore` (dual-licensed under the MIT and Apache licenses) and `libstd` (`libcore`'s licenses, and assorted licenses from `libstd`'s Cargo dependencies), and most real-world binaries directly and transitively depend on tens or hundreds of libraries from Crates.io. The vast majority of that code is made available under licenses that require attribution to the authors and/or reproduction of the appropriate license notice. Manually compiling and maintaining that information is out of the question for pretty much all users.

To my knowledge, there is no tooling that automatically generates the appropriate upstream license file. As a result, I been unable to find a *single Rust project* that appropriately complies with all of its upstream license agreements. This includes:

- Servo
- Ripgrep
- Rustup
- Rustc (includes licenses for C dependencies, but not Rust dependencies)
- Cargo (has LICENSE-THIRD-PARTY file that has not been updated with new dependencies since 2014).

Getting this right is really important, but barely anyone bothers since it's such a massive undertaking. If *Rust's flagship projects* can't get this right, what hope do average users have to confidently manage it correctly without assistance? As such, the standard Rust tooling should include tools that Just Do The Right Thing, so that Rust projects can be fearlessly legally compliant.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

The standard Rust distribution provides tooling to help automatically manage upstream licenses. At a high level, the tooling can be split into two categories: license *specification* and license *retrieval*.

## License Specification

### In Rustc

`rustc --licenses PATH` is the lowest-level brick in the rust licensing toolchain. It takes a path to a JSON-formatted map between licenses, license text, and libraries.

As an example, the following block is a licenses file that contains `serde` and `rand`, used under the Apache license, as well as `syn`, `libc`, and `cfg-if`, used under the MIT license.

```json
{
    "Apache-2.0": {
        "libraries": ["serde", "rand"],
        "text": "{apache license text}"
    },
    "MIT: The Rust Project Developers": {
        "libraries": ["libc"],
        "text": "Copyright 2014 The Rust Project Developers\n\nPermission is hereby granted {...the rest of the MIT license}"
    },
    "MIT: Alex Crichton": {
        "libraries": ["cfg-if"],
        "text": "Copyright 2014 Alex Crichton\n\nPermission is hereby granted {...the rest of the MIT license}"
    },
    "MIT": {
        "libraries": ["syn"],
        "text": "Permission is hereby granted {...the rest of the MIT license}"
    }
}
```

Note how the Apache 2.0 section contains two libraries from two different authors, as the Apache license text does not contain the copyright holder's name, while there are three different MIT sections:
- One [for `libc`](https://github.com/rust-lang/libc/blob/4f11029a68040c90acf771976b019c1ef273a8cd/LICENSE-MIT), under The Rust Project Developers 
- One [for `cfg-if`](https://github.com/alexcrichton/cfg-if/blob/f71bf60f212312faddee7da525fcf47daac66499/LICENSE-MIT), under Alex Crichton
- One [for `syn`](https://github.com/dtolnay/syn/blob/49102f05b3f67f9de3107521080fdbb9f97c00ab/LICENSE-MIT), where the license has no listed copyright holder

Rustc combines this file with a built-in licenses file that includes licenses from Rust's implicit dependencies. For example, if the user were building their crate in a `no_std` context, rustc would combine the user's file with the following file:

```json
{
    "MIT: The Rust Project Developers": {
        "libraries": ["libcore"],
        "text": "Copyright 2014 The Rust Project Developers\n\nPermission is hereby granted {...the rest of the MIT license}"
    }
}
```

Resulting in the following license map (assuming we use the example user-provided license file above):

```json
{
    "Apache-2.0": {
        "libraries": ["serde", "rand"],
        "text": "{apache license text}"
    },
    "MIT: The Rust Project Developers": {
        "libraries": ["libc", "libcore"],
        "text": "Copyright 2014 The Rust Project Developers\n\nPermission is hereby granted {...the rest of the MIT license}"
    },
    "MIT: Alex Crichton": {
        "libraries": ["cfg-if"],
        "text": "Copyright 2014 Alex Crichton\n\nPermission is hereby granted {...the rest of the MIT license}"
    },
    "MIT": {
        "libraries": ["syn"],
        "text": "Permission is hereby granted {...the rest of the MIT license}"
    }
}
```

Rustc then renders the combined file into a human-readable version that can be included in downstream applications. This RFC does not specify how the human-readable version should be formatted. Rustc's built-in file would contain more dependencies when users compile with `libstd`.

### In Cargo

The user generally shouldn't have to hand-write the license file. Instead, it gets automatically generated by Cargo based on the crate's dependency tree and passed into rustc through the standard build process.

If a crate is multi-licensed, Cargo will default to using the first license in the multi-license list. However, users can specify a preferred license via the `[licenses]` section in `Cargo.toml`:

```toml
[licenses]
prefer = ["Apache-2.0", "MIT", "Zlib"]
```

If present, Cargo will prefer the earliest matching license in the `prefer` list. 

If a crate has dependencies with licenses Cargo cannot detect (e.g. in FFI crates), it can specify external licenses via the `licenses.external` field, which contains a path to a JSON licenses file relative to the crate's root:

```toml
[licenses]
external = "./THIRD-PARTY-LICENSES.json"
```

Cargo merges the provided license file with its auto-generated license file before passing the file into rustc.

## License Retrieval

Users can access the license data either directly in the source code or through the filesystem alongside the crate's build artifacts. Both methods are provided so that crate authors can distribute license information as is best suited for their particular application: CLI applications that are distributed via a single binary may want to expose the license information through a command-line argument, while dynamic libraries or applications with more complex distribution methods may want to include the license information as a file that gets distributed alongside the binary.

A compiler built-in macro is provided to retrieve the license information in the source code:

```rust
macro_rules! licenses { () => { /* compiler built-in */ } }
```

`licenses!()` returns a list of all licenses used by the library as an `&'static LicenseList`. `LicenseList` implements `Display` so that you can use `println!` to display all the necessary licenses in human-readable form:

```rust
println!("{}", licenses!());
```

`LicenseList` dereferences to `[License]` to allow you to do more complex structural manipulations on the license list (say, if you'd like to use custom formatting that better suits your application):

```rust
for license in licenses!() {
    println!("# {}", license.name);
    print!("Used by ");
    for library in license.libraries {
        print!("{} ", library);
    }
    println!();
    println!();
    println!("{}", license.text);
}
```

Users can specify `licenses` or `licenses-json` in rustc's `--emit` argument, which outputs the human-readable and JSON-formatted license files to the filesystem alongside the standard build outputs. Cargo passes `--emit licenses` to `rustc` by default, but this can be customized with `cargo rustc --emit`.

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

## License resolution

The license map is formatted as follows: 

```json
{
    "SPDX License Identifier: Optional Copyright Holder": {
        "libraries": ["library0", "library1", "library2"],
        "text": "License Text"
    }
}
```

Keys in the license map should be formatted as a SPDX license identifier (when available), optionally followed by a colon and the copyright holder's name if the license text is customized to each particular copyright holder. This isn't strictly enforced, but is followed by all official tooling. If the license identifier contains a colon, it can be escaped with two consecutive colons (`ANNOYING:IDENTIFIER` -> `ANNOYING::IDENTIFIER`).

The `--licenses` argument is only valid for `crate-type`s that produce a binary file intended for redistribution (`bin`, `dylib`, `staticlib`, and `cdylib`). Other crate types will use the `--licenses` file specified by the nearest parent crate that accepts the `--licenses` flag. Attempting to pass the `--licenses` flag on an invalid crate types will result in an error. This is done to facilitate build artifact sharing. To illustrate, let's say we have a workspace that depends on a `licenses_markdown` crate, which calls `licenses!()` and renders it into a Markdown file. Multiple crates in the workspace depend on this library:

```
workspace_crate_a: cdylib
    licenses_markdown: rlib 
    serde: rlib
    serde_json: rlib
workspace_crate_b: cdylib
    licenses_markdown: rlib
    winit: rlib
    glutin: rlib
```

`workspace_crate_a` and `workspace_crate_b` have different dependency trees, and as such have different license requirements. If the license data were baked into the `licenses_markdown` build artifacts, Cargo would have to recompile the crate whenever it got linked to a different top-level crate. Deferring `licenses` resolution to the higher-level crates allows Cargo to re-use the build artifacts from the lower-level crates when building with different parents. Now, let's say we add an additional crate that linked to `workspace_crate_a`, `workspace_crate_b`, and `licenses_markdown`:

```
workspace_executable: bin
    licenses_markdown: rlib 
    workspace_crate_a: cdylib
        licenses_markdown: rlib 
        serde: rlib
        serde_json: rlib
    workspace_crate_b: cdylib
        licenses_markdown: rlib
        winit: rlib
        glutin: rlib
```

`licenses!()`, as invoked for `workspace_crate_a` and `workspace_crate_b`, would continue to only contain the licenses they directly depend on (so, `workspace_crate_a` would not include a license for `winit`). However, `workspace_executable` would include licenses from the entire dependency tree, regardless if they were behind a `cdylib` or not. Bear in mind, if the `workspace_crate`s were `rlib`s, all invocations of `licenses!()` would return the same value.

---

Cargo splits the license string into multiple licenses on any of the following patterns:
- `/`
- `,`
- `\bOR\b`

Additional patterns may be added as discovered in the ecosystem.

Cargo will attempt to discover license files in the crate's root folder. Cargo will pull the `LICENSE` or `COPYRIGHT` file for single-licensed crates, and will make a best-effort attempt to match the license in multi-licensed crates with the following rules by finding the file in the crates root that best matches the particular license's SPDX identifier. This draft currently doesn't define the exact file matching rules, but the full RFC should probably be more specific here.

## The `license!()` macro

The `licenses!()` macro is added to `libcore`, and returns a `&'static LicenseList`. A new `licenses` module is added to `libcore`, containing the following types:

```rust
#[derive(Debug, PartialEq, Eq, Hash, PartialOrd, Ord)]
pub struct LicenseList {
    licenses: [License]
}

impl Deref for LicenseList {
    type Target = [License];
    // implementation
}

impl Display for LicenseList {
    // renders the complete human-readable license list
}

#[derive(Debug, Clone, Copy, PartialEq, Eq, Hash, PartialOrd, Ord)]
pub struct License {
    pub name: &'static str,
    pub libraries: &'static [&'static str],
    pub text: &'static str,
}

impl Display for LicenseList {
    // renders the license in human-readable form
}
```

We return `&'static LicenseList` instead of `&'static [License]` so that the `Display` trait can be implemented on `licenses!()`'s return type. The data returned by the macro is adapted from the license data passed into `rustc` via the `--licenses` flag.

# Drawbacks
[drawbacks]: #drawbacks

More complexity in the language infrastructure. Most languages don't seem to provide a license management mechanism built into the compiler, but most languages don't make it as easy to add new dependencies as Rust does.

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

## Why is this built into the language?

It's entirely valid to ask, "why are we making this a compiler feature instead of a toolchain feature?". It's not immediately clear that that's the best option; license information is admittedly metadata and you can reasonably argue that this process should be entirely handled by Cargo and related tools. This RFC involves the compiler for two reasons:

- `#![no_std]` is a language-level attribute, not a Cargo-level attribute. There needs to be *some* process for determining whether or not `libstd`'s licenses should be included, and Cargo is currently unable to derive this information.
- The `licenses` macro cannot exist without rustc's involvement. If this were a Cargo-level process, it'd still be possible to embed license information into the crate with build scripts, but that's significantly more finnicky and less easy-to-use than a language-level macro.

Given those points, this RFC decides that it would be easiest to include license processing in the compiler as well as in the Cargo infrastructure. It's worth noting that Cargo may eventually be able to manage `std` - the [`cargo-std-aware` WG](https://github.com/rust-lang/wg-cargo-std-aware) is making progress on that front - but that work will likely take years to complete and this RFC is written for the language as it exists today.

## Alternatives

- Let the community manage this, and provide a stable way to access the licenses implicitly used by the standard libraries.
- Standardize `Cargo.toml` fields for specifying all necessary license information, and let third-party crates handle compiling that information into a usable form.
- Instead of having `licenses!()` return a structure, we could have the macro return a rendered license string.
- Instead of specifying dependency licenses in the top level of the build tree, we could have each crate pass the license information the crate is responsible for into `rustc` and include it in the generated `rlib`. This might actually be the better solution, but I'm not including it as the primary solution in this draft because it's unclear to me how this would interact with `dylib`s and `cdylib`s. Worth discussing more thoroughly.
- Instead of passing license information via a command-line argument, we could pass it via an environment variable.

# Prior art
[prior-art]: #prior-art

Mozilla Firefox has shell scripts that automatically generate a human-readable license file: https://github.com/mozilla/application-services/blob/master/DEPENDENCIES.md. This isn't entirely complete, as it doesn't contain licenses for Rust's `libstd` or `libcore`, but it's the best I've seen.

----

Various crates exist in the ecosystem for listing and managing licenses
- https://crates.io/crates/cargo-license
- https://crates.io/crates/cargo-deny

Both crates provide utilities for listing upstream licenses, and `cargo-deny` provides utilities for rejecting incompatible licenses (say, the GPL). However, neither crate peeks into `libstd` or `libcore`, and neither crate automatically assembles a human-readable license file.

# Unresolved questions
[unresolved-questions]: #unresolved-questions

- We need to figure out the exact process Cargo uses to resolve license files.
- What should we do when a crate specifies a license in Cargo.toml, but doesn't include the license file in the Cargo package?
- What should we do when crate's don't specify a license?

# Future possibilities
[future-possibilities]: #future-possibilities

- This RFC has cargo choose the first license in the multi-license list so that it doesn't have to be aware of license semantics. In theory, it could automatically select the most permissive license, for whatever definition of permissive Cargo chooses to accept.
- `[licenses]` could have a disallowed licenses list that warns if a forbidden license is detected.
- Cargo could automatically warn upon license incompatibilities.
