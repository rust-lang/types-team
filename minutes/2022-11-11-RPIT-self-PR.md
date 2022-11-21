# Behaviour of inherited lifetime parameters for opaque types

Zulip discussion link: https://rust-lang.zulipchat.com/#narrow/stream/326132-t-types.2Fmeetings/topic/2022.E2.80.9011-11.20PR.20review/near/309197876
PR link: https://github.com/rust-lang/rust/pull/103491

The current implementation of async and RPIT replace all lifetimes from the parent item's generics by `'static`.
The point of this scheme is to ensure that only the lifetimes that explicitly appear in the bounds for the opaque:
- are allowed to occur inside the opaque (while borrow-checking the defining function);
- are considered to occur inside the opaque (while borrow-checking the user functions).

However, using `'static` creates unwanted behaviour when `Self` and associated type projections appear in bounds.
In pseudo syntax, the desugaring looks like this:
```rust
trait Trait<'x> {
    type Assoc;
}

impl<'a> Foo<'a> {
    fn foo<'b, 'c, 'd, T: Trait<'b>>() -> impl Into<Self> + From<T::Assoc> + 'c { ... }
}

opaque type Foo::<'_a>::foo::<'_b, '_c, '_d, T>::opaque<'c>:
    Into<Foo<'_a>> // `Self` is implicitly desugared to mention `'_a`
  + From<<T as Trait<'_b>>::Assoc> // `T::Assoc` is implicitly desugared to mention `'_b`
  + 'c; // `'_d` stays unused, as expected

impl<'a> Foo<'a> {
    fn foo<'b, 'c, T: Trait<'b>>() -> Foo::<'static>::foo::<'static, 'static, 'static, T>::opaque::<'b> { ... }
}
```

This implies that the bounds are:
```rust
opaque type Foo::<'static>::foo::<'static, 'static, 'static, T>::opaque<'c>:
    Into<Foo<'static>> // `Self` gets the wrong lifetime
  + From<<T as Trait<'static>>::Assoc> // The trait bound is wrong too
  + 'c;
```

In order to avoid accepting wrong code and silently transmuting a `Foo<'a>` to a `Foo<'static>`,
`Self` and projections were banned from impl-trait.

# Proposed resolution

The point of the original device was to ensure that `'_a`, `'_b`, `'_c` and `'_d` did not appear in borrowck,
but just the duplicated `'c` version. A more "proper" desugaring would look like:

```rust
impl<'a> Foo<'a> {
    fn foo<'b, 'c, 'd, T: Trait<'b>>() -> Foo::<'a>::foo::<'b, 'c, 'd, T>::opaque::<'c> { ... }
}
```

The issue becomes: how to tell borrow-checking that the opaque only actually depends on `'_a`, `'_b` and `'c` but not `'_d`.

We want to:
- only relate substs that correspond to captured lifetimes during TypeRelation;
- only list captured lifetimes in "occurs check".

We propose to encode usage/non-usage using `variances_of` query:
`Bivariant` vs `Invariant` for unused vs captured lifetimes.

Impl-trait that do not reference `Self` or projections will have their variances as:
- `o` (invariant) for each type or const generic param;
- `*` (bivariant) for each lifetime param from parent --> will not participate in borrowck;
- `o` (invariant) for each own lifetime param.

Impl-trait that do reference `Self` and/or projections will have some parent lifetimes marked as `o` as they appear in the bounds, and participate in type relation and borrowck.

In the example above, `'_a` and `'_b` appear in bounds, so: `variances_of(opaque) = ['_a: o, '_b: o, _c: *, '_d: *, T: o, 'c: o]`.

# Unresolved questions

Is this even sound?

Is there a lighter way to do that, instead of changing `TypeRelation`?

Should we insert a `'_c == 'c` bound somewhere? --> yes

Variance computation for opaques is not integrated into the crate-global variance computation:
1. it makes conservative assumptions about mutually recursive TAITs;
2. bivariance does not leak into ADTs.

# Comments

oli:
Why do we need the additional lifetimes on the opaque type? Can't we just mark the right ones as invariant/bivariant?

cjgillot: to  support late-bound lifetimes in functions `fn foo(&self) -> impl Debug + '_`

Less of a question: I'd like to land this first for TAIT and add tests that ensure that TAIT works exactly like RPIT. Then we can migrate RPIT without any language concerns.

## AFIT interaction

nikomatsakis: This problem is related to AFIT somehow, right? Can we say a bit more about the connection?

compiler-errors: https://github.com/rust-lang/rust/issues/102681, basically `Self` type ends up being substituted with `'static` types, which causes the method signature comparison is `compare_predicate_entailment` to unnecessarily relate a bunch of non-`'static` lifetimes with `'static`

## wellformedness interactions

nikomatsakis: I am not sure if this proposal is addressing the interesting problem. In the past when we talked about it, we had settled on the need for some kind of existential lifetime support. The reason was because of cases like this, which is currently accepted:

```rust
trait Foo<'a>: Bar { }

trait Bar { }

fn foo<'a, T>(x: T) -> impl Debug
where
    T: Foo<'a>,
{ 
    MyStruct { x }
}

struct MyStruct<T: Bar> { x: T }

impl<T> Debug for MyStruct<T>
where
    T: Bar,
{
    fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
        panic!()
    }
}
```

Note that `'a` is not captured, but for the `MyStruct<T>` to be WF, we must know that `T: Bar`, and we know that because we know that `T: Foo<'a>` for *some* `'a`. This works today but because of some rather hacky things.

There are also other examples, I'm trying to find the hackmd where we went through the various cases -- there doesn't have to be a supertrait relationship.

## variance

nikomatsakis: I don't love using variance to encode this, though it also doesn't seem entirely wrong.

## oli alternative

nikomatsakis: Didn't oli have some other approach to this problem? I don't know what it was, though.
