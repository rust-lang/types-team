# 2022-07-01 Planning meeting

Zulip discussion link: https://rust-lang.zulipchat.com/#narrow/stream/326132-t-types.2Fmeetings/topic/meeting.202022-07-01/near/288033700

## Initiative updates

### NLL

* Centralized on the NLL checker
* Found several unsound situations in how NLL was handling higher-ranked projects, some fixed, some not yet:
    * #98095: Outlives requirements for projections, `for<'a> T: 'a` in where-clauses. Fixed by XXX.
    * #98693: Promoting requirements in higher universes. Fixed by #98713 (not yet landed).
    * #98589: Investigated -- not a *soundness* concern per se, but more of a "surprising behavior" concern, where the inferencer seems to be introducing inference that allows for named lifetimes in a closure to be approximated if it would be otherwise sound to do so. Should probably fix to prevent code from relying on it.

**Question:** Should we beta revert?

*Situation:* Stabilized NLL. Found 1 bug (#98095) 

*Premise:* if we do so, we'll give 6 more weeks to find bugs on nightly

*But:*

* Existing bugs are all concerning treatment of higher-ranked regions, which makes sense as that is where lexical check still differed from NLL
* Most have been found by systematically comparing NLL code -- that process has completed, but we can do another check
* Fixes for existing bugs have not had broad ecosystem impact (so unlikely to affect a lot of people)
* Reverting on beta is a lot of work

*Ways to reduce risk:*

* We've not finished investigating #98589, so we can't say with certainty the impact of a fix there. We'll try to prioritize that.
* Do a final review of the NLL code, given the existing pattern of bugs, to look for new instances

### GATs

* Posted summary comments:
    * [Summarizing concerns](https://github.com/rust-lang/rust/pull/96709#issuecomment-1129311660)
    * [Deep-dive into some use patterns](https://github.com/rust-lang/rust/pull/96709#issuecomment-1167220240)
* Other things:
    * Bevy report coming
    * Niko [synced with lang team to start FCP](https://github.com/rust-lang/lang-team/blob/master/minutes/2022-06-28.md#generic-associated-types)
    * Niko diving into WF stuff

### TAITs

* [unsoundness due to lazy TAIT refactor](https://github.com/rust-lang/rust/issues/98608)
    * minimal paper-over PR available, but is perf regression
    * working on a real fix
* [allow destructuring opaque types](https://github.com/rust-lang/rust/pull/98582)
    * `let (a, b) = some_tait;` registers a tuple as the hidden type instead of ICEing
* [wf check generators](https://github.com/rust-lang/rust/pull/97183)
    * crater clean, needs review
* [wf check opaque types](https://github.com/rust-lang/rust/pull/95474)
    * blocked on t-types checky boxes in https://github.com/rust-lang/rust/pull/97406

### RPIT refactor

- We are lowering APITs and impl/traits blocks generics from AST, so no HIR copying.
- Fixed some minor bugs related to the feature flag handling and some async fns issues
- There are 3 issues that I've identified already:
  - in bound normalization tests where some lifetime bounds are not properly mapped to the RPIT, need to investigate this better
  - the RPIT doesn't have the Sized bound added implicitly, so if nothing in the function has that bound the RPIT is not Sized which is wrong 
  - Some const fn issues in async functions, tries to subst &[] with index 0
- Maybe @nikomatsakis/@oli can explain better than me the 'static problem

### Subtyping refactor

* Remove universe logic from NLL
    - current step is:
        - simplify the "outlives requirement" (`T: 'a'`) clauses in the trait solver, not in the region check
        - https://github.com/rust-lang/rust/pull/97641/
    - next step is:
        - make outlives environment available to trait solver, ideally refactor into parameter environment
        - we used to have to wait until type inference was done, but now (in NLL) we don't have to
    - blocked on:
        - https://github.com/rust-lang/rust/pull/98584
    - subsequent steps is:
        - handle type parametrs and regions in fulfill by searching parameter bounds etc
        - we can remove the "outlives" code entirely
        - and NLL no longer has to think about higher-ranked stuff
            - can remove universes from NLL* niko to think about how new subtyping algorithm coallesces of lifetime edges and model in a-mir-formality
* no progress on splitting predicates into goals and clauses

### Trait object upcasting

In a [recent Zulip thread](https://rust-lang.zulipchat.com/#narrow/stream/144729-t-types/topic/dyn-upcasting.20stabilization), there was a long conversation about stabilization, resulting in:

* [Revised document](https://hackmd.io/@nikomatsakis/rJDzKH-Fq) outlining the various options. This resulted in a simpler set of choices:
    * "Fully valid vtable", which requires a fully valid vtable to create a `*const dyn`. We established that some of the more complex framings of this solution (e.g., "sufficiently valid vtables") weren't really necessary to buy forwards compatibility. This option is maximally conservative, but can be loosened in the future.
    * "Operations are UB", which permits creating `*const dyn` values with arbitrary vtables, but makes it UB to upcast or invoke methods on them, which effectively forbids them from escaping to safe code (since those operations are safe). This option is maximally permissive for users in terms of avoiding UB.

The next step here is to check in briefly with unsafe code guidelines WG representatives and move to stabilization with one of the above variants.

### Negative impls

Current state is that the code for this is largely implemented, modulo one outstanding issue ([#93875](https://github.com/rust-lang/rust/issues/93875)).

We added a [rough model of coherence](https://github.com/nikomatsakis/a-mir-formality/pull/66) to a-mir-formality.

No notable progress on the [RFC](https://hackmd.io/ZmpF0ITPRWKx6jYxgCWS7g?both).

### a MIR Formality

### Polonius

* Developed a [roadmap and task listing](https://hackmd.io/@nikomatsakis/rJDoTg9Y5?type=view#task-listing)
* upcoming tasks:
    * drafting canonical loan analysis, ideally in mir-formality

### chalk-ty

* We are now unblocked to move 
* [Moved `RegionKind` to `rustc_type_ir`](https://github.com/rust-lang/rust/pull/98247)
* @eggyal continues to make progress on aligning TypeFoldable/TypeVisitor
* Needs review:
    * #98206
    * chalk#772

## Nominated issues

### #98117 Unsoundness due to where clauses not checked for well-formedness

### #98543 implied bounds from associated types may not actually get implied pt 2.

### #98693 NLL and closures: higher-ranked lifetime bounds are not enforced in type tests