# deep dive 9/9/22

[zulip meeting](https://rust-lang.zulipchat.com/#narrow/stream/326132-t-types.2Fmeetings/topic/2022-09-09.20projection.20equality)

Note: this doc often refers to a `ProjectionEq` obligation, this is actually called `ProjectionPredicate` in rustc apparently which seems like kind of a silly name to me. Apologies for constantly messing up the name in this doc and likely anywhere I talk about this lol

# projection equality status quo

currently relating projections happens "eagerly", for example `<_ as Trait>::Assoc eq u32` fails because we can't wait around to see if the projection normalizes to `u32`.

## Other type is not a projection

If we relate `<_ as Trait>::Assoc eq SomeOtherType<...>` it will currently fail which is incorrect as it may later end up with an inferred self type which allows for normalization, i.e. `<u64 as Trait>::Assoc`.

This currently is mitigated by sometimes normalizing `<_1 as Trait>::Assoc` to some new inference var `_2` and emitting a `ProjectionEq` obligation. However this falls short for higher ranked stuff which can be gotten from where clauses i.e. `where for<'a> <T as Trait<'a>>::Assoc: Other`, causing issues [#96230](https://github.com/rust-lang/rust/issues/96230) and [#89196](https://github.com/rust-lang/rust/issues/89196).

## Other projection is a different `DefId`

If we relate:
`<_1 as Trait>::Assoc eq <_2 as Trait>::Assoc2`
currently it would fail because the projections are for different defids. This is overly restrictive as `_2` might end up getting resolved to some concrete type `T` allowing the projection to be normalized to: 
`<T as Trait>::Assoc`
which would then be able to succeed in being equated with:
`<_1 as Trait>::Assoc`

## Relating substs

If we relate `<u32 as Trait<_1>>::Assoc eq <u64 as Trait<_1>>::Assoc` currently it would fail because `u32 eq u64` does not hold. This is overly restrictive is the impl `u32: Trait<_1>` and `u64: Trait<_1>` may have the same value for the `type Assoc`.

Similarly if we relate `<_1 as Trait>::Assoc eq <_2 as Trait>::Assoc` we would constrain `_1 eq _2` which may not necessarily be required and later cause type inference to error because `_2` turned out to be a `u32` and `_1` turned out to be a u64.

# deferred projection equality

PR [#96912](https://github.com/rust-lang/rust/pull/96912) changes the logic of relating projections to (if the projection has infer vars) succeed and emit a `ProjectionEq` obligation. This seems to fix all of the above issues (surprisingly as I was only aiming to fix the first issue).

As of writing this the PR updates `ProjectionEq` obligation handling to only relate the substs and defid of a `ProjectionTy` if it is "as normalized as it will ever get". I am not sure what precise terminology for this is but the idea is that `<T as Trait>::Assoc` is never going to be able to normalized more, but a `<_ as Trait>::Assoc` might.

Fixing "relating substs" is not entirely backwards compatible as now relating `<_ as Trait>::Assoc eq <T as Trait>::Assoc` will leave the inference var unconstrained, it would be simple to "unfix" this in the PR.

Note: the exact code changes in [#96912](https://github.com/rust-lang/rust/pull/96912) are a bit :grimacing: right now. specifically `project.rs` and winnowing, they certainly will be changed before seriously attempting to land the PR. (More about the winnowing changes below.)

# #96912 issues

## typenum regression

below is a minimal repro of the code that fails in typenum with the lazy projection equality PR, annotated with the candidates/obligations that are being evaluated during winnowing:

```rust
#![recursion_limit = "4"]
#![allow(unused_must_use)]

use std::ops::Shl;

pub struct Foo<U>(U);

pub trait NotSub: Sized {
    type Output;
    fn make_output(&self) -> Self::Output;
}

impl<A, B> Shl<Foo<B>> for Foo<A>
where
    Foo<B>: NotSub,
    Foo<Foo<A>>: Shl<<Foo<B> as NotSub>::Output>,
{
    type Output = ();
    fn shl(self, rhs: Foo<B>) {
        let lhs: Foo<Foo<A>> = Foo(self);
        let rhs: <Foo<B> as NotSub>::Output = <Foo<B> as NotSub>::make_output(&rhs);
        // <
        //      Foo<Foo<A>>
        //      as
        //      Shl<
        //          <Foo<B> as NotSub>::Output
        //      >
        // >::shl(lhs, rhs)
        <_ as Shl<_>>::shl(lhs, rhs);
        //  obligation: Foo<Foo<A>> as Shl<_#0t>
        //  candidates:
        //  - param `Foo<Foo<A>> as Shl<<Foo<B> as NotSub>::Output>`
        //      - ???
        //      - ???
        //  - impl `for<A2, B2> Foo<A2> as Shl<Foo<B2>>`
        //      - requires _#0t == Foo<_#1t>
        //      - substs: [Foo<A>, _#1t]
        //      - nested:
        //          - Foo<_#1t>: NotSub
        //              - requires _#1t == B
        //          - Foo<Foo<Foo<A>>>: Shl< <Foo<_#1t> as NotSub>::Output >
        //            candidates:
        //              - impl `for<A2, B2> Foo<A2> as Shl<Foo<B2>>`
        //                  - substs: [Foo<Foo<A>>, _#2t]
        //                  - nested:
        //                      - ProjectionEq(<Foo<_#1t> as NotSub>::Output == Foo<_#2t>)
        //                      - Foo<_#2t>: NotSub
        //                          - requires _#2t == B
        //                      - Foo<Foo<Foo<Foo<A>>>>: Shl< Foo< _#2t > as NotSub >::Output >
        //                          - same as previous `Foo<Foo<Foo<A>>>` obligation but with an extra `Foo` this recurses infinitely
    }
}
```

The comments in the code shows all the candidates for each obligation being solved and the nested obligations from each candidate. When evaluating the `impl` candidate we end up evaluting infinitely recursing `Foo<..<..<Foo<A>..>..>: Shl< <Foo<_> as NotSub>::Output >` obligations.

In order to make the code compile we need rustc to understand that the `impl` candidate doesn't apply and the `param` candidate does. The impl candidate can `EvaluateToErr` if the obligations are evaluated in a specific order:
- `Foo<_#1t>: NotSub` this unifies `_1 eq B`
- `Foo<Foo<Foo<A>>>: Shl< <Foo< _1 > as NotSub>::Output >`
- `ProjectionEq(<Foo<B> as NotSub>::Output == Foo<_2>)` note that this is actually `Foo<_1> as NotSub` but we have unified `_1 eq B` which allows us to know that this `ProjectionEq` obligation does not hold.

Currently we do not even evaluate the `ProjectionEq` obligation, we evalaute `Foo<Foo<Foo<Foo<A>>>>: Shl< Foo<_2> as NotSub >::Output` first and then `Foo<Foo<Foo<Foo<Foo<A>>>>>: Shl<...>` etc etc. This causes us to get a recursion limit error which ends compilation rather than an `EvaluatedToErr` which would let us reject the impl candidate and move on to successfully evaluating the `param` candidate. 

We could do a lot better at avoiding these recursion errors by removing `evaluate_predicate_recursively` and using `ObligationCtx` (honestly I don't know what type to use but lcnr mentioned using `ObligationCtx` so...) to evaluate candidates. This would ensure that evenf if `ProjectionEq` was evaluated and then `Foo<_1>: NotSub` was evaluated it would be alright since the `ProjectionEq` obligation could stall on `_1`. Using an `ObligationCtx` also removes the possibility of us recursing forever without evaluating some obligations (This is more of an assumption, I don't actually know enough to know if this is completely true...).

An alternative way to make this specific code compile would be to take advantage of the fact that impl candidates get discarded in favor of param candidates. We could potentailly evaluate all param candidates first and if any are `EvaluatedToOk`, discard all the impl candidates without evaluating them. There is a perf run for this solution at [#97380](https://github.com/rust-lang/rust/pull/97380).

The PR currently implements the latter solution (evaluate param candidates first). It would not be backwards compatible to remove this and replace it with only fulfill like evaluation as `ObligationCtx` does not prevent us from ever getting a recursion limit error. Using `ObligationCtx` seems like a nicer solution and something that would be good to do anyway so that we don't have to maintain two separate ways of evaluating obligations.

## projection eq variance

if we have a `<_ as Trait>::Assoc sub Foo<'a>'` and later `_` gets inferred to `Foo<'b>` which allows us to normalize the projection to `Foo<'b>` then we would end up doing `Foo<'b> eq Foo<'a>` not `Foo<'b> sub Foo<'a>`.

# Questions

## How do I ask a question

Like this.

## @lcnr: nested fulfillment during evaluation

`evaluate_predicate_recursively` sucks, using a new fulfillment context there instead should work but allows some code to compile which may break again when going "full chalk". How does it work in chalk rn?

nikomatsakis (spying in, not 100% here) -- I think that in chalk-style solver we would evaluate both paths but discard the impl path (eventually) as ambiguous -- but probably not until types got too large or something, I have to look more closely at that.

## implications merging 97380

> what are the implications of just merging this on its own? It looks like a perf improvement even with the further-improvable implementation.
- oli

codes kinda janky and would not be entirely safe to remove later