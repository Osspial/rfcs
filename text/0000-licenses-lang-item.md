- Feature Name: licenses_lang_item
- Start Date: (fill me in with today's date, YYYY-MM-DD)
- RFC PR: [rust-lang/rfcs#0000](https://github.com/rust-lang/rfcs/pull/0000)
- Rust Issue: [rust-lang/rust#0000](https://github.com/rust-lang/rust/issues/0000)

# Summary
[summary]: #summary

- Add two new macros:
    + `licenses!()`: a human-readable list of all in-project and upstream licenses.
    + `licenses_json!()`: a JSON-formatted list of all in-project and upstream licenses.
- Add to the `rustc` CLI:
    + `rustc --emit licenses`: emit the human-readable license list as part of the compiler output
    + `rustc --emit licenses-json`: emit the JSON-formatted license list as part of the compiler output
    + `rustc --licenses PATH`: path to a JSON-formatted list of the licenses used by user-compiled code
- Add to `cargo`
    + `cargo licenses --format FMT   Licenses format: human,json [default: human]` Prints the license information generated by rustc to `stdout`
    + `[licenses]` section in `Cargo.toml`: configure how Cargo generates the license list passed to `rustc`
- Add `--help --licenses --format FMT   Licenses format: human,json [default: human]` to all standard Rust CLI applications.

# Motivation
[motivation]: #motivation

Every single piece of code in the Rust ecosystem, from a simple "Hello, World" example to a complex application with hundreds of dependencies through Cargo, depends on licensed open-source software. At the lowest level, projects are going to depend on `libcore` (dual-licensed under the MIT and Apache licenses) and `libstd` (`libcore`'s licenses, and assorted licenses from `libstd`'s Cargo dependencies), but most real-world binaries directly and transitively depend on tens or hundreds of libraries from Crates.io. The vast majority of that code is made available under licenses that require attribution to the authors and reproduction of the appropriate license notice. Manually compiling and maintaining that information is out of the question for pretty much all users.

To my knowledge, there is no tooling that automatically generates the appropriate upstream license file. As a result, I been unable to find a *single Rust project* that appropriately complies with all of its upstream license agreements. This includes:

- Servo
- Rustc
- Ripgrep
- Rustup
- Cargo (has LICENSE-THIRD-PARTY file that has not been updated with new dependencies since 2014).

Getting this right is really important, but barely anyone even tries since it's such a massive undertaking. As such, the standard Rust tooling should include tools that Just Do The Right Thing, so that Rust projects can be fearlessly legally compliant.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

The standard Rust distribution provides tooling to help automatically manage upstream licenses. At a high level, the tooling can be split into two categories: license *specification* and license *retrieval*.

## License Specification

`rustc --licenses PATH` is the lowest-level brick in the rust licensing toolchain. It takes a path to a JSON-formatted map between licenses, license text, and libraries. The map is formatted as follows: 

```json
{
    'SPDX License Identifier: Optional Copyright Holder': {
        'libraries': ['library0', 'library1', 'library2'],
        'text': 'License Text'
    }
}
```

As an example, the following block is a licenses file that contains `serde` and `rand`, used under the Apache license, as well as `libc` and `cfg-if`, used under the MIT license.

```json
{
    'Apache-2.0': {
        'libraries': ['serde', 'rand'],
        'text': '{apache license text}'
    },
    'MIT: The Rust Project Developers': {
        'libraries': ['libc'],
        'text': 'Copyright 2014 The Rust Project Developers\n\nPermission is hereby granted {...the rest of the MIT license}'
    },
    'MIT: Alex Crichton': {
        'libraries': [`cfg-if`],
        'text': 'Copyright 2014 Alex Crichton\n\nPermission is hereby granted {...the rest of the MIT license}'
    }
}
```

Note how the Apache 2.0 section contains two libraries from two different authors, as the Apache license text does not contain the copyright holder's name, while there are two different MIT sections for each copyright holder.

Rustc combines this file 

Explain the proposal as if it was already included in the language and you were teaching it to another Rust programmer. That generally means:

- Introducing new named concepts.
- Explaining the feature largely in terms of examples.
- Explaining how Rust programmers should *think* about the feature, and how it should impact the way they use Rust. It should explain the impact as concretely as possible.
- If applicable, provide sample error messages, deprecation warnings, or migration guidance.
- If applicable, describe the differences between teaching this to existing Rust programmers and new Rust programmers.

For implementation-oriented RFCs (e.g. for compiler internals), this section should focus on how compiler contributors should think about the change, and give examples of its concrete impact. For policy RFCs, this section should provide an example-driven introduction to the policy, and explain its impact in concrete terms.

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

Keys in the map should be formatted as a SPDX license identifier, optionally followed by a colon and the copyright holder's name if the license text is customized to each particular copyright holder. This formatting isn't strictly enforced, but is followed by all of the official Rust tooling.


This is the technical portion of the RFC. Explain the design in sufficient detail that:

- Its interaction with other features is clear.
- It is reasonably clear how the feature would be implemented.
- Corner cases are dissected by example.

The section should return to the examples given in the previous section, and explain more fully how the detailed proposal makes those examples work.

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

# Future possibilities
[future-possibilities]: #future-possibilities

Think about what the natural extension and evolution of your proposal would
be and how it would affect the language and project as a whole in a holistic
way. Try to use this section as a tool to more fully consider all possible
interactions with the project and language in your proposal.
Also consider how the this all fits into the roadmap for the project
and of the relevant sub-team.

This is also a good place to "dump ideas", if they are out of scope for the
RFC you are writing but otherwise related.

If you have tried and cannot think of any future possibilities,
you may simply state that you cannot think of anything.

Note that having something written down in the future-possibilities section
is not a reason to accept the current or a future RFC; such notes should be
in the section on motivation or rationale in this or subsequent RFCs.
The section merely provides additional information.