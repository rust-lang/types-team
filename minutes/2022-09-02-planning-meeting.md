# 2022-09-02 planning meeting

## Proposed meetings

### The meeting proposals

* [discuss projection equality](https://github.com/rust-lang/types-team/issues/51)
* [discuss the trait_alias feature](https://github.com/rust-lang/types-team/issues/49)
* [Variance and Rust](https://github.com/rust-lang/types-team/issues/45)
* [Polonius status update](https://github.com/rust-lang/types-team/issues/43)
* [Finalize recommendations for TAITs](https://github.com/rust-lang/types-team/issues/40)
* [Types team roadmap](https://github.com/rust-lang/types-team/issues/53)
* [Chalk integration plan](https://github.com/rust-lang/types-team/issues/54)
* [RPIT refactor review](https://github.com/rust-lang/types-team/issues/55)

Proposed meetings:
* Sep 9 -- projection equality (types-team#51)-- @**Boxy [she/her]**
* Sep 16 -- finalize recommendations for TAITs (types-team#40) -- @**oli**
* Sep 23 -- RPIT refactor review (types-team#55) -- @**Santiago Pastorino**
* Sep 30 -- chalk integration plan (types-team#54) -- @**nikomatsakis** / @**Jack Huey**

## Status updates

### RPIT refactor

We've fixed yesterday the last issue and we will probably be able to open a PR today with 2 diagnostics regressions, which I think we can open issues about and land this as is.

:tada: --nikomatsakis --oli

### TAITs

* https://github.com/rust-lang/rust/pull/95474 (don't imply `impl Trait<'a>: 'a`)
    * real world breakage on crater found (3 crates)
    * proactively provided PRs to the broken projects
    * will fix one significant diagnostic regression before merging
    * re-FCP with T-types?
* https://github.com/rust-lang/rust/pull/99806 (allow `let (x, y) = some_opaque_value` to be a defining use)
    * perf hit, still analyzing the cause
* https://github.com/rust-lang/rust/pull/98933
    * just needs a rebase and some cleanup work

leftover issues before merging: https://github.com/rust-lang/rust/issues?q=is%3Aopen+is%3Aissue+label%3AF-type_alias_impl_trait+assignee%3Aoli-obk

### GATs

Is in FCP as of Wednesday. 8 days left. Should make it into 1.65. Notes from lang meeting: https://github.com/rust-lang/lang-team/blob/master/design-meeting-minutes/2022-08-31-gat-stabilization.md

GATs GATs GATs woooo --oli

### a-mir-formality

* merged voidc's rustc front end, so that we can generate tests from rust source code
* implemented the start of borrow checker -- in particular, liveness and initialization checks
* spastorino landed some improvements to coherence checker and is investigating modeling negative impls

### NLL

Done? Should WG get closed?

### Subtyping refactor

Goal is to have the traits/subtyping code manage HRTB. As part of that, lcnr has been working on moving implied bounds into the parameter environment:

> Moving implied bounds into the `ParamEnv` required a general refactoring to how implied bounds are implemented. The first step for this was https://github.com/rust-lang/rust/pull/100676 which changes rustc to eagerly compute implied bounds. This PR accidentally exacerbated an existing soundness bug https://github.com/rust-lang/rust/issues/100051, causing me to revert that accidental behavior change in https://github.com/rust-lang/rust/pull/100989.
>
> I am now close to finishing the move of implied bounds into the `ParamEnv`. This cleanup also makes it a lot easier to fix https://github.com/rust-lang/rust/issues/100051 and will allow us to consider more types for implied-bounds.

### Trait object upcasting

* we had some discussion in [dyn-upcasting stabilization](https://rust-lang.zulipchat.com/#narrow/stream/144729-t-types/topic/dyn-upcasting.20stabilization). Key conclusions:
    * No-op trait upcasts don't count, we are only interested in upcasts that change the "principal trait(s)" (i.e., affect the methods that can be invoked, and hence alter the vtable). This was formalized in https://github.com/rust-lang/rust/pull/100208.
    * We prefer the "operations are UB" variant as [described in this document](https://hackmd.io/z9GaT_vdRtazrmcF6pStvQ) as it sems maximally consistent.
        * One minor open question is whether vtables ought to be defined as aligned pointers to create a niche for "raw wide pointers", e.g., `*const dyn Foo`. There is a niche today (but not for `*const T` where `T: Sized`).
        * Observation: backwards compatible to remove the niche.

### Negative impls

* spastorino landed https://github.com/rust-lang/rust/pull/100888, closing the last known issue in the implementation
* rfc draft has not made any progress

### Polonius

* discussed roadmap with Amanda Stjerna and Giacomo Pasini [on zulip](https://rust-lang.zulipchat.com/#narrow/stream/186049-t-compiler.2Fwg-polonius/topic/2022-08)
    * Giacomo is going to investigate a proposed change to MIR with the goal of eliminating duplicate logic between borrow checker and drop elaboration, and giving a single unified version of MIR for all static analyses. nikomatsakis needs to write up that proposal as an MCP and discuss with oli and MIR optimization team.

### chalk-ty

No progress. Well, small PR: https://github.com/rust-lang/rust/pull/100095

