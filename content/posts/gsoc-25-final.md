+++
title = "GSoC 2025 Final Report"
date = "2025-08-29"

[taxonomies]
categories = ["en"]
tags = [
    "Rust",
    "Compiler",
    "GSoC",
]
+++


_I love making holes in stuff._

## Overview
[My project](https://summerofcode.withgoogle.com/programs/2025/projects/3y9x5X8O) this summer is about `stable_mir`, a crate that is being developed to be the public API of the Rust compiler.
Things in `stable_mir` have been changed a lot so far. We even [renamed](https://rust-lang.zulipchat.com/#narrow/channel/320896-project-stable-mir/topic/Renaming.20StableMIR/near/527945657) `stable_mir` to a cooler name: `rustc_public`, which better reflects the project's scope, because nowadays our APIs coverage goes beyond [MIR](https://rustc-dev-guide.rust-lang.org/mir/index.html), and we don't aim to provide stability in the common sense, e.g., Rust 1.0, that is not what we aim at.

The high-level goal for this summer is to prepare the `rustc_public` (a.k.a. `stable_mir`) crate for publishing. Speaking of which, we aim to publish this crate on crates.io, allowing users to explicitly select it.

## My work
### Refactoring the crates structure
The previous crate structure is the main blocker for publishing. For context, there were two crates in the Rust compiler: `stable_mir` and `rustc_smir`. We intend to invert the dependency between them, making `rustc_smir` completely agnostic of `stable_mir`. For the full background and the reasons why we want to do that, please refer to [this proposal](https://hackmd.io/@celinaval/H1lJBGse0) for more details. Here I'd like to directly walk through the PRs related to this work.

Firstly, given that our ulitmate goal is to reverse the dependency order, which should be accomplished with a sequence of PRs, we have to figure out the strategy to resolve the circular dependency that would occur during the development process. For some reason the word "parasitism" came to mind: what if we make `stable_mir` parasitic on `rustc_smir`? Therefore:
- [rust-lang/rust#139319](https://github.com/rust-lang/rust/pull/139319)
This PR moved `stable_mir` into the `rustc_smir` crate as a module, which was a temporary tweak to resolve the [circular dependency](https://en.wikipedia.org/wiki/Circular_dependency) that would arise if we directly invert the dependency order between `rustc_smir` and `stable_mir`.

Okay, now we can freely feed `stable_mir` to grow up. What's next? It is:

#### Implement `CompilerInterface`
- [rust-lang/rust#139852](https://github.com/rust-lang/rust/pull/139852)
This PR split the previous `Context` struct into two parts: `SmirInterface` and `SmirCtxt`. See [this section](https://hackmd.io/@celinaval/H1lJBGse0#Examples) for more details. After this PR was merged, our code was like:
```rust
// stable_mir/compiler_interface.rs
/// Stable public API for querying compiler information.
///
/// All queries are delegated to an internal [`SmirCtxt`] that provides
/// similar APIs but based on internal rustc constructs.
///
/// Do not use this directly. This is currently used in the macro expansion.
pub(crate) struct SmirInterface<'tcx> {
    pub(crate) cx: SmirCtxt<'tcx>,
}

impl<'tcx> SmirInterface<'tcx> {
    /// Obtain the representation of a type.
    pub(crate) fn ty_kind(&self, ty: Ty) -> TyKind {
        self.cx.ty_kind(ty)
    }
}


// rustc_smir/context.rs
/// Provides direct access to rustc's internal queries.
///
/// The [`crate::stable_mir::compiler_interface::SmirInterface`] must go through
/// this context to obtain rustc-level information.
pub struct SmirCtxt<'tcx>(pub RefCell<Tables<'tcx>>);

impl<'tcx> SmirCtxt<'tcx> {
    /// Obtain the representation of a type.
    pub fn ty_kind(&self, ty: stable_mir::ty::Ty) -> TyKind {
        let mut tables = self.0.borrow_mut();
        tables.types[ty].kind().stable(&mut *tables)
    }
}
```
This was getting close to what we anticipated. xD

#### Refactor StableMIR
- [rust-lang/rust#140643](https://github.com/rust-lang/rust/pull/140643)
This was a huge PR that completed most of the refactoring and introduced many implementation details. I don't want to list them one by one here. By the way, the most painful part of this work was manually [rewriting](https://github.com/rust-lang/rust/pull/140643/files#diff-610545890aa062638aa875d81f8f07280e92c113b5df9f34ffdff6e29b487ce0) nearly a hundred APIs .

Now that we've finished the refactoring, let's move it back!
- [rust-lang/rust#143524](https://github.com/rust-lang/rust/pull/143524)
This PR moved `stable_mir` back to its own crate.

Finally, we renamed `stable_mir` to `rustc_public`, and `rustc_smir` to `rustc_public_bridge`.
- [rust-lang/rust#143848](https://github.com/rust-lang/rust/pull/143848)
- [rust-lang/rust#143985](https://github.com/rust-lang/rust/pull/143985)
- [rust-lang/rust#144290](https://github.com/rust-lang/rust/pull/144290)

At the end of the day, the code above now looks like:
```rust
// rustc_public/compiler_interface.rs
impl<'tcx> CompilerInterface for Container<'tcx, BridgeTys> {
    /// Obtain the kind of a type.
    fn ty_kind(&self, ty: Ty) -> TyKind {
        let mut tables = self.tables.borrow_mut();
        let cx = &*self.cx.borrow();
        cx.ty_kind(tables.types[ty]).stable(&mut *tables, cx)
    }
}


// rustc_public_bridge/context/mod.rs
/// Provides direct access to rustc's internal queries.
///
/// `CompilerInterface` must go through
/// this context to obtain internal information.
pub struct CompilerCtxt<'tcx, B: Bridge> {
    pub tcx: TyCtxt<'tcx>,
    _marker: PhantomData<B>,
}


// rustc_public_bridge/lib.rs
/// A container which is used for TLS.
pub struct Container<'tcx, B: Bridge> {
    pub tables: RefCell<Tables<'tcx, B>>,
    pub cx: RefCell<CompilerCtxt<'tcx, B>>,
}
```

### Establishing infrastructure
For context, we'd maintain two versions of `rustc_public`, the crates.io version and the rustc version. The first will be published on crates.io and follow semantic versioning, while the second will track internal changes and serve as the base for each new major release.

For this to happen, we leverage [rustc-josh-sync](github.com/rust-lang/josh-sync) to sync changes as we see fit. We've cloned `rustc_public` into our own repo in [rust-lang/project-stable-mir#104](https://github.com/rust-lang/project-stable-mir/pull/104). We'd enable the josh cronjob in [rust-lang/project-stable-mir#107](https://github.com/rust-lang/project-stable-mir/pull/107).

#### Devtool
- [rust-lang/project-stable-mir#105](https://github.com/rust-lang/project-stable-mir/pull/105)
This is a development helper for maintaining `rustc_public`. Currently it supports commands like `build`, `test [--bless] [--verbose]`, `clean`, `fmt [--check]`, `githook install/uninstall`, and `help`. To stay consistent with the rustc development experience, I named our shell script x, so contributors can simply run commands like `./x build` or `./x test`.

#### Build-time check
- also in [rust-lang/project-stable-mir#105](https://github.com/rust-lang/project-stable-mir/pull/105)
I write a simple `build.rs` to warn the user using an unsupported rustc version, make the build fail with emitting some suggestions.

#### Compiletest
- also in [rust-lang/project-stable-mir#105](https://github.com/rust-lang/project-stable-mir/pull/105)
This is a test script leveraging the [`ui_test`](https://github.com/oli-obk/ui_test/) crate to run all test suites we now have.

That said, by the time I wrote this sentence, [rust-lang/project-stable-mir#105](https://github.com/rust-lang/project-stable-mir/pull/105) was still under review.

### Misc
- Removed a FIXME from `rustc_public`: [rust-lang/rust#144392](https://github.com/rust-lang/rust/pull/144392).
- Fixed a missing parenthesis in pretty printing: [rust-lang/rust#145119](https://github.com/rust-lang/rust/pull/145119)
- Fix book deployment CI: [rust-lang/project-stable-mir#106](https://github.com/rust-lang/project-stable-mir/pull/106)
- Fix the rustc_public demo: [rust-lang/project-stable-mir#108](https://github.com/rust-lang/project-stable-mir/pull/108)


## Future work

### Release automation
I was planning to implement a release automation CI, 

### Documentation

### A stable rustc_public driver

## Thanks to
- My mentor Celina Val, who’s been very patient and helped me learn a lot.
- oli-obk, who suggested me to set up the josh sync, and reviewed dozens of my PRs.
- Deadbeef :3
- Jakub Beránek, who helped me learn more about the josh.
- Jieyou Xu, I don't know but I just want to say thanks :3
- the infra team.
- everyone that I've chatted with on Zulip so far.
- the Rust Foundation.
- Google, definitely.