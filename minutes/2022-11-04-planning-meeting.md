# 2022-11-04 planning meeting

Zulip discussion link: https://rust-lang.zulipchat.com/#narrow/stream/326132-t-types.2Fmeetings/topic/2022-11-04.20planning/near/307951923
Previous planning meeting: https://hackmd.io/VV4OlAgKQXKKxqI8dbWMvA

## Meeting agenda

* Fill in status updates
* Review and schedule proposed meetings
* Topics for extended discussion
    * Proposed workflow:
        * create tracking issues for each item and give them a tag (e.g., `I-types-team-initiative`).
        * bot to scrape repos for issues with that agenda and include recent updates
        * ability to ping bot and have it ping people assigned to those issues

## Proposed meetings

* Variance and Rust #45
* discuss the trait_alias feature #49
* Types team roadmap #53
    * Done? In progress?

## Status updates

### RPITIT refactoring

It's almost done.

* We are already lowering the RPIT as an associated type in both the trait and the corresponding impl.
* I've gone over the involved queries adjusting them and making them do the right thing.
* There are currently 15 failing test cases, from the ones I remember ...
    * trait with default bodies
    * async fns
* I'm currently going over project, candidate assembly, confirm and friends in order to better understand how they work.

### AFIT/RPITIT stabilization

They work surprisingly well, the only existing issues are due to `'static` in the impl-trait desugaring causing some outlives issues when comparing methods' compatibility with their traits.
A few folks are already using AFIT experimentally:
* Embedded HAL: https://github.com/rust-embedded/embedded-hal/pull/407
* Yosh's async_iterator crate: https://docs.rs/async-iterator/2.0.0/async_iterator/trait.Iterator.html

Existing blockers due to `'static` substitution in lifetimes causing strange lifetime equality bugs, e.g.: https://github.com/rust-lang/rust/issues/102681
cjgillot's PR https://github.com/rust-lang/rust/pull/103491 attempts to fix this, otherwise associated types are quite limited in AFIT.
* this PR could use a deep-dive, since cjgillot told me that they're uncertain if it's sound.

Open issue for auto trait bounds issue: https://github.com/rust-lang/rust/issues/103854
Though definitely requires some lang design work that doesn't necessarily block static AFITs landing, since all changes would be forward-compatible.
* compiler-errors might put up a PR for `T: Trait<fn(): Send>` type bounds, since they're not that hard to hack into the compiler.

### TAITs

TAIT is just one merge away from being ready for stabilization. I will write a doc now explaining what TAIT does and what it doesn't.

### GATs

Stable! (Onwards and upwards to the next thing (working on implied static))

### a-mir-formality

Working on rust-rewrite in the riir branch. Things are progressing well. Hoping to open it up for others to contribute to soon. Current status is that end-to-end tests kind of work, but there are still a few basic pieces in the trait solver (e.g., outlives relationships) to finish modeling before it's really ready to be scaled out. Using Rust is nice.

### Subtyping refactor

No progress on this directly.

nikomatsakis: I believe we can meet the major goal of unblocking polonius in a simpler way. Essentially moving the higher-ranked code that we use *today* in NLL into a pre-pass on the constraints coming out from the trait solver. This item could perhaps be removed from the list.

### Implied bounds

WIP PR ready except for inductive trait solver cycles due to the additional WF bounds. Avoiding them in a non hack-ish way will probably require us to change the solver to be coinductive for all traits.

### Non-fatal trait solver overflow

Requires changes to the while API of the trait solver, so it's a good work to start the rustc trait solver rewrite initiative with :sparkles: the current API surface is pretty massive and they tend to need slightly different approaches. Making progress on this though and am not stuck on anything yet.

### Trait object upcasting

[RFC #3324](https://github.com/rust-lang/rfcs/pull/3324) is in FCP, finishes in a day or two, and then we should be ready to stabilize. :tada:

### Negative impls

Tracking issue: https://github.com/rust-lang/rust/issues/68318

No major progress here, though nikomatsakis has been thinking about the issue of cycles (https://github.com/rust-lang/rust/issues/102678) and plans to model a fix in a-mir-formality. Good first goal for the riir branch, actually. Santiago and he should touch base on the idea, which could also be implemented in rustc.

Other blockers:

* https://github.com/rust-lang/rust/issues/79098 
* 

### Polonius

Had a hackathon meeting yesterday with a number of people. Went back over the rules and identified two main next steps:

* Implementation: plan is to explore an implementation in rustc that builds on NLL infrastructure. lqd is leading this, with theo plus some others interested. lqd + nikomatsakis plan to meet once more to do high-level design.
* Rule design: we've got several variants of the rules by now and need to get to one set. nikomatsakis was drafted rules in a-mir-formality but paused that work for the riir branch. Domenic Quirl is going to be doing some detective work on the various alternatives. Another meeting needs to be scheduled here too to go over the results. 

### chalk-ty
