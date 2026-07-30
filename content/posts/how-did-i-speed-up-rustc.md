+++
title = "How to make rustc 94% faster (in some cases)"
date = "2026-07-30"

[taxonomies]
categories = ["en"]
tags = [
    "Rust",
    "rustc",
    "Compiler",
    "derive macros"
]
+++

For those who suspect the title is a scam, I'm going to destroy your question with this chart:

<div align="left"><img src="/how-to-speed-up-rustc/summary.png" width="100%" alt="summary.png" title="hi"></div>

- You can find and reproduce the result [here](https://codeberg.org/makai410/debug-fmt-perf/src/branch/blog-reprod).

As you can see the test case is a file with 100 enums, each of which has over 8001 variants, which is apparently fairly common
in the real-world rust code and you've probably written some code like this before, haven't you?

## So... then?

So the story is about `derive(Debug)` expansion which is not **that trivial**.
The first implementation of `derive(Debug)` in rustc before *Jan 21, 2023, 10:43 UTC* was like this:
```rust
#[derive(Debug)]
pub enum Fieldless0 {
    A,
    BBB,
    CC,
}

#[automatically_derived]
impl ::core::fmt::Debug for Fieldless0 {
    fn fmt(&self, f: &mut ::core::fmt::Formatter) -> ::core::fmt::Result {
        match self {
            Fieldless0::A => ::core::fmt::Formatter::write_str(f, "A"),
            Fieldless0::BBB => ::core::fmt::Formatter::write_str(f, "BBB"),
            Fieldless0::CC => ::core::fmt::Formatter::write_str(f, "CC"),
        }
    }
}
```

It looks... very straightforward! Here we have a match where each arm corresponds a function call. This is suboptimal since
every arm makes its own inline call to `write_str()`. That doesn't matter *that much* for small enums. For a large one,
though, all those calls add up to a bloated MIR body. Its MIR looks like this:
```
// simplified
bb2: {
    _10 = &mut (*_2);
    _12 = const "CC";
    _11 = &(*_12);
    _0 = Formatter::<'_>::write_str(move _10, move _11) -> bb8;
}

bb3: {
    _7 = &mut (*_2);
    _9 = const "BBB";
    _8 = &(*_9);
    _0 = Formatter::<'_>::write_str(move _7, move _8) -> bb8;
}

bb4: {
    _4 = &mut (*_2);
    _6 = const "A";
    _5 = &(*_6);
    _0 = Formatter::<'_>::write_str(move _4, move _5) -> bb8;
}

bb8: {
    return;
}
```
- [godbolt link](https://godbolt.org/z/nzYnfYbqW)

The obvious fix is to hoist the `write_str` out of the match. Each arm then only has to select a string, leaving a single call
to `write_str` in the generated function. This makes the MIR body much smaller. [^1]
This was done in [rust-lang/rust#106884](github.com/rust-lang/rust/pull/106884) by [@clubby789](https://github.com/clubby789) with a great perf win✌️.

Now the second version of `derive(Debug)` expansion looks like this:
```rust
#[derive(Debug)]
pub enum Fieldless0 {
    A,
    BBB,
    CC,
}

#[automatically_derived]
impl ::core::fmt::Debug for Fieldless0 {
    fn fmt(&self, f: &mut ::core::fmt::Formatter) -> ::core::fmt::Result {
        ::core::fmt::Formatter::write_str(f, match (self) {
            Fieldless0::A => "A",
            Fieldless0::BBB => "BBB",
            Fieldless0::CC => "CC",
        })
    }
}
```
Yes, looks so good so far, doesn't it? We often wish some stories could end at a point when we're content with how things are and pray that things 
won't go wrong or stray too far from our expectations. Nonetheless, usually, they head in a direction we'd never expect,
yet somehow feel inevitable in hindsight.

## The big one is coming

Consider [the case where we have an enum with 132 variants](https://github.com/rust-lang/rust/issues/114106). Now you can see the problem. Hoisting `write_str` doesn't really
patch the one-match-arm-per-variant hole, since the generated function still grows along with the enum:

```rust
#[derive(Debug)]
pub enum Errno {
    UnknownErrno,
    EPERM,
// ...other 130 variants
}

#[automatically_derived]
impl ::core::fmt::Debug for Errno {
    fn fmt(&self, f: &mut ::core::fmt::Formatter) -> ::core::fmt::Result {
        ::core::fmt::Formatter::write_str(match (self) {
            Errno::UnknownErrno => "UnknownErrno",
            Errno::EPERM=> "EPERM",
            // ...other 130 variants
        })
    }
}
```
- from [the nix crate](https://docs.rs/nix/latest/nix/errno/enum.Errno.html)

## Lookup table comes to the rescue!...?
There was already a lookup table solution to the problem, i.e.
we create an array crammed with all the string literals and index into it with the discriminant.
While the array still grows with the enum, of course, rustc doesn't need to generate one match arm per variant.
This way we *might* have even better performance, but the caveat is that this requires the discriminants to be dense.
```rust
impl ::core::fmt::Debug for Fieldless0 {
   fn fmt(&self, f: &mut ::core::fmt::Formatter) -> ::core::fmt::Result {
       ::core::fmt::Formatter::write_str(f, {
           const __NAMES: [&str; 3] = ["A", "BBB", "CC"];
           __NAMES[::core::intrinsics::discriminant_value(self) as usize]
       })
   }
}
```

This attempt was implemented in both [rust-lang/rust#109615](https://github.com/rust-lang/rust/pull/109615) and [rust-lang/rust#114190](https://github.com/rust-lang/rust/pull/114190),
while both showed a *regression* in the perf results.

I think directly looking at the generated MIR could help triage the problem.
And I have to say, the moment I look at it, I found three problems [^2]:
- every 
- [godbolt](https://godbolt.org/z/nvaWza9WE)



[^1]: [rust-lang/rust#106875](https://github.com/rust-lang/rust/issues/106875)
[^2]: [“我一进店就发现三个问题” ("The moment I walked into the shop, I found three problems.")](https://www.bilibili.com/video/BV11hjh6gEVS/?share_source=copy_web&vd_source=928d170c8684db8a8c05fb13fb92bd23)

first impl of concatenated string: https://godbolt.org/z/8vhYxo9b7
