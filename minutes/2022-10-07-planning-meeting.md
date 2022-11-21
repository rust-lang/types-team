# 2022-10-07 planning meeting

Zulip discussion link: https://rust-lang.zulipchat.com/#narrow/stream/326132-t-types.2Fmeetings/topic/2022-10-07.20Planning.20meeting/near/302846151
Previous planning meeting: https://hackmd.io/BN-RofYeTZKKc_5Iu3sPFg

## Proposed meetings

## Status updates

### RPIT refactor

* PR got closed :(
* Changing to RPIT/RPITIT cleanup

### TAITs

* landed https://github.com/rust-lang/rust/pull/95474
* opened https://github.com/rust-lang/rust/pull/102417
* opened https://github.com/rust-lang/rust/pull/102700
* Status tracked here: https://github.com/orgs/rust-lang/projects/22/views/1

### GATs

* Stable now ~~?~~ !
* TODO: blog post for 1.65 release
* Issues tracked here: https://github.com/orgs/rust-lang/projects/17
    * Some, at least
* Adding `PredicateTy` to rustc: https://github.com/jackh726/rust/tree/a-whole-new-world

### a-mir-formality

voidc landed some improvements to the type checker.

Currently working on adding a version of the borrow checker (modeling NLL, to start). This has led me to realize that we *may* be able to make a lightweight change to rustc that would help us with "problem case #3":

```rust
if let Some(v) = self.map.get() {
    return v;
}
self.map.insert();
```

Currently investigating if this is true. :) 

Going to be giving a talk on mir-formality next week at HILT '22. Will share slides when they're worth looking at.

### Subtyping refactor

mostly distracted by opaque types in ctfe. annoying :angry: 

### Trait object upcasting

Blocked on the need to author an RFC. Should do that.

### Negative impls

Did preliminary integration into a-mir-formality and uncovered a problem, filed at [#102678], concerning cyclic reasoning. The basic idea is that, before we rely on `(T: !Foo) => (T: Foo)`, we need to establish that. The planned fix is to refactor coherence and leverage query system to avoid cycles.

[#102678]: https://github.com/rust-lang/rust/issues/102678

### Polonius

We are planning a series of hackathons to get oriented, starting Oct 26. We will be adding these to the compiler team calendar.

### chalk-ty

* No `TypeFoldable` for `EarlyBinder`: https://github.com/rust-lang/rust/pull/101901
