# TAITs ahoy

Zulip meeting link: https://rust-lang.zulipchat.com/#narrow/stream/326132-t-types.2Fmeetings/topic/2022-09-16.20TAIT.20stabilization.20finalizations/near/299140021

* [Old project board](https://github.com/rust-lang/wg-traits/projects/4)
* [New project board](https://github.com/rust-lang/impl-trait-initiative/projects/2)
* [tracking issue](https://github.com/rust-lang/rust/issues/63063)

## 2022-09-16 - Status Update T-types Deep Dive

Full status board: 
https://github.com/orgs/rust-lang/projects/22/views/1

* [patterns constraining hidden type is a perf regression](https://github.com/rust-lang/rust/issues/96572)
    * `(x, y) = some_value_of_opaque_type` errors very unhelpfully
    * can fix after stabilization, but not great
* [assoc TAITs are useless for "common" use cases](https://github.com/rust-lang/rust/issues/95922)
    * fixed by chalk/implied bounds

Just waiting on https://github.com/rust-lang/rust/pull/98933 and https://github.com/rust-lang/rust/pull/95474 to get merged so the last known soundness bugs in TAITs are solved.

### Open question: Should TAITs capture all lifetimes?

Original issue: https://github.com/rust-lang/rust/issues/96996

```rust
#![feature(type_alias_impl_trait)]
type StrRef<'a> = impl AsRef<str>;
fn define(s: &str) -> StrRef<'_> { s }
```

compiles, even though there is no `+ 'a` bound on `impl AsRef<str>`. This is not what the RFC specified, and is different from how RPIT works. It is often convenient, as you don't have to use the `+ Captures<'a>'` trick, and instead all lifetimes of the type alias, assoc type or trait are captures.

Workarounds exist, so we may just say, that if you don't want to capture all lifetimes, use a free TAIT without the lifetimes you want to capture: https://github.com/rust-lang/rust/issues/91601

Will this be a problem for nested `impl Trait` capturing HKL? I guess we could also use the workaround where this is a problem.

So this is mainly a language question: do we want implicit capture or be in sync with RPIT and require explicit capture?

### Where are we at now?

Type alias impl trait is the feature where you can use `impl Trait` syntax in the type of a type alias or an associated type:

```rust
type Foo<'a> = (i32, impl Bar);
impl Moo for Meh {
    type Boo<T> = Vec<impl Bar>;
}
```

These uses of `impl Bar` each generate a unique type (an opaque type) that has all the same generics (including lifetimes) as the type alias/assoc type. So effectively the above desugars to

```rust
opaque type X<'a>: Bar;
type Foo<'a> = (i32, X<'a>);

opaque type Y<U, T>: Bar;
impl<U> Moo for Meh {
    type Boo<T> = Vec<Y<U, T>>;
}
```

It is possible to replace any use of `impl Trait` in return types with a type alias:

```rust
fn foo<'a, 'b, T>() -> impl Trait + 'a {}
```

can be replaced with


```rust
type Foo<'a, T> = impl Trait;
fn foo<'a, 'b, T>() -> Foo<'a, T> {}
```

without any changes in behaviour. The advantage of using type aliases is that you can re-use the same opaque type in multiple locations, and guarantee to the user that they are the same type.

```rust
type Foo<'a, T> = impl Trait;
fn foo<'a, 'b, T>() -> Foo<'a, T> {}
fn bar<'x, T>() -> Foo<'x, T> {}

let mut x = foo();
x = bar(); // both return the same type, so this compiles.
```

The type-alias/assoc-type nature has a few effects that may be surprising when compared with other types:

### No implied oulives bounds

In contrast to the same program if `Foo` were a struct, this does not compile:

```rust
type Foo<'x> = impl Debug;
fn foo<'a, 'b>(x: &'a Foo<'b>) -> &'b i32 {
    unimplemented!()
}
```

[playground](https://play.rust-lang.org/?version=nightly&mode=debug&edition=2021&gist=59e657a4aa7deba979bc830340e4933e)

`Foo<'b>` not add any constraints on `'b`, because hidden types don't actually need to reference the lifetimes on the opaque type. Thus they can live arbitrarily longer. The following program would be unsound if we allowed such implications.

```rust
type WithLifetime<'a> = impl Sized;
fn _defining_use<'a>() -> WithLifetime<'a> {}

trait Convert<'a> {
    type Witness;
    fn convert<'b, T: ?Sized>(_proof: &'b Self::Witness, x: &'a T) -> &'b T;
}

impl<'a> Convert<'a> for () {
    type Witness = WithLifetime<'a>;

    fn convert<'b, T: ?Sized>(_proof: &'b WithLifetime<'a>, x: &'a T) -> &'b T {
        // cannot imply `'a: 'b` from `&'b WithLifetime<'a>`
        x
    }
}
```

### No inference across separate items

Inference of opaque types only happens within the same function. The following program errors because `foo` infers the type to be `i32` via fallback, while `bar` sets the type to `i64`

```rust
type Foo = impl Debug;
fn foo() -> Foo { 42 }
fn bar() -> Foo { 69_i64 }
```

### Inference at a distance via return types

Opaque types in return types allow type inference between all return sites (no matter if via `return`, `?` or via a trailing expression).

```rust
fn foo(b: bool) -> impl Debug {
    if b {
        return vec![42];
    }
    some_iterator.collect()
}
```

### Not only return sites are constraining hidden types

You can constrain a hidden type in any direction. This means you can also take an opaque type and turn it into a concrete type.

```rust
type Foo = impl Debug;
fn foo(x: Foo) {
    let y: i32 = x;
    let z = y + 1;
}
```

### Fields of hidden types can be revealed within the same function

More spooky action at a distance that is actually just following the regular inference logic. If you moved the `if false` block later in the function, the code would not compile as it would not know the type of `y.1` and thus not be able to call `push`.

```rust
type Foo = impl Debug;
fn foo(x: Foo) {
    if false {
        // this sets the type
        let a: Foo = (42_i32, String::new());
    }
    
    let y: (_, _) = x;
    // can access the types here.
    let z = y.0 + 1;
    let mut s = y.1;
    s.push('X');
}
```

### type alias impl trait can be used anywhere another type can

```rust
type Foo = impl Debug;

struct Bar {
    x: Foo,
}

const MOO: Foo = unimplemented!();

fn bar() -> Bar {
    Bar {
        x: "hello",
    }
}
```

### Questions and Notes

> So this is mainly a language question: do we want implicit capture or be in sync with RPIT and require explicit capture?

nikomatsakis: I think we need to have a way to have explicit capture, but I've become slowly converted to the POV that this may not be `type Foo = impl Trait`. Specifically, I think it might make sense to have this syntax:

```rust
type Foo<'a>: Trait;
```

and then have `type Foo<'a> = impl Trait` desugar to

```rust
type Foo<'a> = Foo1;
type Foo1: Trait;
```

just as with RPIT (in this case, then, the `'a` would not be captured in the TAIT notation, only in the explicit opaque type notation). One advantage to this distinction is that you can move a function return type into a type alias and the semantics stay the same:

```rust
fn foo<'a, T>(x: &'a T) -> (impl Display, impl Display + 'a) { (22, x) }

type Foo<'a, T> = (impl Display, impl Display + 'a);
fn foo<'a, T>(x: &'a T) -> Foo { (22, x) }
```

It creates (to my eye) a nice parallelism between associated types, and it addresses boats' concern that "TAITs desugar to themselves". Ultimately, though, I think this is a lang-team call, but it makes sense for us to have a recommendation.

I wonder if we could do some targeted experiments to try and measure learnability. I'm going to think about this.

**Meeting discussion:** [Discussed here](https://rust-lang.zulipchat.com/#narrow/stream/326132-t-types.2Fmeetings/topic/2022-09-16.20TAIT.20stabilization.20finalizations/near/299160911) with the [meeting consensus](https://rust-lang.zulipchat.com/#narrow/stream/326132-t-types.2Fmeetings/topic/2022-09-16.20TAIT.20stabilization.20finalizations/near/299163800) that we ought to open a write-up for lang team and put the following question:

* today's behavior, where TAITs capture all lifetimes *or*
* TAITs behave like RPITs with respect to lifetime capture, and we debate a future "opaque type" syntax

with our recommendation being option 2.

---

> It is possible to replace any use of `impl Trait` in return types with a type alias \[..\] without any change in behavior.

Really? Doesn't that affect the scope of constraining uses, which in turn may make a different in some cases? 

**Meeting discussion:** [Discussed here](https://rust-lang.zulipchat.com/#narrow/stream/326132-t-types.2Fmeetings/topic/2022-09-16.20TAIT.20stabilization.20finalizations/near/299164426) and concluded that it is equivalent, but only if you add modules and things to control the scope.

---

> "TAIT: it's possible to impl a trait for a tait by using projections" https://github.com/rust-lang/rust/issues/99840

unsoundness in the project board, not mentioned here:

issue is that coherence uses https://doc.rust-lang.org/nightly/nightly-rustc/rustc_infer/infer/enum.DefiningAnchor.html#variant.Error while it should treat opaque types as ambiguous. At this point we should also allow explicitly mentioning the opaque type in impl headers. Not really a question, just the statement: "we should also allow explicitly mentioning the opaque type in impl headers"

**Meeting discussion:** [Discussed here](https://rust-lang.zulipchat.com/#narrow/stream/326132-t-types.2Fmeetings/topic/2022-09-16.20TAIT.20stabilization.20finalizations/near/299164603) and concluded that we should prohibit TAITs from inherent impls, but that we will have to accept them in trait impls, because they can be named indirectly (e.g., through projections). The plan is to treat them equivalently to a projection (i.e., "some unknown type") which should be sound with respect to coherence.

---

> Inference at a distance via return types

Does this *not* work for TAITs, then? i.e., this is an error?

```rust
type Return = impl Debug;

fn foo() -> Return {
    if something {
        None // type is `Option<_>`
    } else {
        Some(22) // type is `Option<i32>`
    }
}
```

edit: rereading the section, I don't believe it is trying to say that. In particular, the example showing "action at a distance" inference effects suggests that all "proposed values" for the hidden type within a single function are unified, and eagerly so:

```rust
type Foo = impl Debug;
fn foo(x: Foo) {
    if false {
        // this sets the type
        let a: Foo = (42_i32, String::new());
    }
    
    let y: (_, _) = x;
    // can access the types here.
    let z = y.0 + 1;
    let mut s = y.1;
    s.push('X');
}
```

**Meeting discussion:** [Discussed here](https://rust-lang.zulipchat.com/#narrow/stream/326132-t-types.2Fmeetings/topic/2022-09-16.20TAIT.20stabilization.20finalizations/near/299165812), along with the next question (see below).

---

What's the reason behind constraining hidden types at use site? This allows casting from a hidden type to a more specific type (e.g. `impl Debug` to i32), which might be accidental if you write the use site before the implementation.

```rust
type Foo = impl Debug;
fn foo(x: Foo) {
    let y: i32 = x;
    let z = y + 1;
}
```

**Meeting discussion:** [Discussed here](https://rust-lang.zulipchat.com/#narrow/stream/326132-t-types.2Fmeetings/topic/2022-09-16.20TAIT.20stabilization.20finalizations/near/299165812), along with the previous question (see below). We discussed how this formulation can be useful and how it is hard to separate unifications in cases like these from other cases, as well as how to think about the previous question. The short version is that all constraints within a single function are combined with one another, and that in the return type in particular we replace opaque types with an inference variable which can lead them to influence coercions (this came up later on another question).

---

> Trait matching on opaque types

The text did not describe the distinction between a variable of type `Foo` and the hidden type, but I believe it remains significant, right? In other words, given a TAIT...

```rust
type Foo = impl Debug;
```

...if I have a local variable of type `Foo`, I cannot assume anything but that it implements `Debug`...

```rust
let x: Foo = 22_i32;
println!("{x}"); // Error: `x` not known to implement `Display`
```

...but if I use the hidden type, I can...

```rust
let x: Foo = 22_i32;
let x: i32 = x;
println!("{x}"); // OK!
```

It occurs to me that if we had `impl Trait` in let bindings (soon!), then this behavior is easier to explain, since you can first show...

```rust
let x: impl Debug = 22;
```

...which, to my eyes, reads like we are *intentionally* limiting ourselves to `Debug` (and that is the way to think of `let x: Foo` as well, I think).

(Side note, are we intended to make the two entirely analogous? i.e., a `let x: impl Trait` is just a TAIT scoped to the function? I think that's probably the best thing.)

**Meeting discussion:** [Discussed here](https://rust-lang.zulipchat.com/#narrow/stream/326132-t-types.2Fmeetings/topic/2022-09-16.20TAIT.20stabilization.20finalizations/near/299167802), we concluded that this description is correct, and that we don't really have to decide what `let x: impl Trait` will do, but desugaring to a TAIT within the function body is the most likely behavior.

---

From zulip:

> one question: I remember there was some discussion a while back about, well, I'm not sure, a regression or something. I don't see that listed here. I guess I can go find the issue number.

oli replied "we reverted that and are sticking to the existing behaviour", but what was the change and problem? I never understood it. --nikomatsakis

**Meeting discussion:** [Discussed here](https://rust-lang.zulipchat.com/#narrow/stream/326132-t-types.2Fmeetings/topic/2022-09-16.20TAIT.20stabilization.20finalizations/near/299168690). The reversion had to do with the way that RPIT can sometimes take advantage of things that are known from the function context which would not otherwise be known. We uncovered an [example where extracting an RPIT to a TAIT caused errors](https://rust-lang.zulipchat.com/#narrow/stream/326132-t-types.2Fmeetings/topic/2022-09-16.20TAIT.20stabilization.20finalizations/near/299170531) as a result and are considering the best fix.

---

nikomatsakis: we should talk about a-mir-formality and what this looks like! I'll give that some thought. I think we're at the point where we could plausibly model this.

**Meeting discussion:** None really. :) 

## 2022-05-12

### Issues that need decision

https://github.com/rust-lang/rust/issues?q=is%3Aopen+label%3AI-needs-decision+label%3AF-type_alias_impl_trait
    
* [rust-lang/rust#96572](https://github.com/rust-lang/rust/issues/96572): TAIT: "unconstrained opaque type" error even if it's constrained
    * MIR does not see the tuple type at the point of the move. It just sees accesses to the fields.
    * HIR typeck is fine, but MIR-typeck has a fit.
    * if you do not supply a type, then we do not borrowck (which is what infers the hidden type), and so we do not have a hidden type, and thus you end up with the "unconstrained opaque type" error.
    * to fix this, would need to either add some sort of "reveal opaque type" operation, or teach the projection code that if it has an opaque type and is trying to get a field, then it needs to reveal the opaque type.
    * or we could try to make this a more explicit error, one that points at the `let (a, b) = foo;`, since that is the code that is actually injecting the problem here.
    * longer term, we would want to put in an actual fix, but for the short term, try to put the explicit error in, preferably before stabilization so that people do not hit such ICE's on the stable form of the feature.
* [rust-lang/rust](https://github.com/rust-lang/rust/issues/96552): "Experiment: try out removing opaque types from typeck expectations"
    * oli tried this. it makes TAIT behave like RPIT. But it also breaks some (unstable) cases.
    *
    stops compiling now
    ```rust
    // src/test/ui/type-alias-impl-trait/closures_in_branches.rs
    type Foo = impl std::ops::FnOnce(String) -> usize;
    fn foo(b: bool) -> Foo {
        if b {
            // below is new error
            |x| x.len() //~ ERROR type annotations needed
        } else {
            panic!()
        }
    }
    ```
    starts compiling now
    ```rust
    // src/test/ui/lazy-type-alias-impl-trait/branches.rs
    type Foo = impl std::fmt::Debug;

    fn foo(b: bool) -> Foo {
        if b {
            vec![42_i32]
        } else {
            // below error goes away
            std::iter::empty().collect()
            //~^ ERROR `Foo` cannot be built from an iterator over elements of type `_` 
        }
    }
    ```
    * with this change, we unify the collect expression with the other branch of the `if`, allowing it to figure out that it must be `Vec`. Without this change, the collect expression is unified with the opaque return type, instead of unifying with the other branch of the `if`.
    * some cases started compiling, which is the I-needs-decision motivator.

```rust
type Foo = impl std::fmt::Debug;

fn foo(b: bool) -> Foo {
    if b {
        vec![42_i32]
    } else {
        // below error goes away
        std::iter::empty().collect() // Here: the return type is inferred to be the opaque type `Foo`
        // collect requires that `Foo: FromIterator<Item = ?T>`
        // but the *opaque type* does not 
        //
        // separately, we unify `Foo` and `Vec<i32>`
    }
}

fn bar() {
    let b: impl Iterator<Item = u32> = ...;
    let b: Foo = ...;
}

fn bar() {
    let b: Foo = 42;
    let c = b + 5; // ERROR (intuitive?)
}


fn bar() -> Foo {
    let b: Foo = vec![42];
    if something {
        vec![22]
    } else {
        vec![]
    }
}

fn bar() -> Foo {
    let b: Foo = vec![42];
    if something {
        b
    } else {
        std::iter::empty().collect()
    }
}

```

```rust
// src/test/ui/type-alias-impl-trait/closures_in_branches.rs
type Foo = impl std::ops::FnOnce(String) -> usize;
fn foo(b: bool) -> Foo {
    if b {
        // below is new error
        |x| x.len() //~ ERROR type annotations needed
    } else {
        panic!()
    }
}
```

Conclusion for https://github.com/rust-lang/rust/issues/96552:

* We should not clear the expectations
* Rather we should eagerly replace TAITs that appear in the Return Position with inference variables, like we do with RPIT
    * This brings a more consistent model for users: people who read from a place typed with a TAIT are limited to its declared traits
    * But when you store into it, you are not; returning is arguably a store
        * recursive calls, on the other hand, are reads, and will be limited appropriately


The following errors on stable, there is no fallback inside other types

```rust
fn weird() -> PhantomData<impl Sized> {
    PhantomData //~^ ERROR type annotations needed
}
```

but there is fallback if it is just a plain impl Trait

```rust
fn weird() -> impl Sized {
    // ok
}
fn weird() -> impl Sized {
    panic!() //ok, falls back to unit with RPIT, `!` with TAIT (on master today)
    // the new proposal would change this to `()` also with TAIT
}
```

```rust=
type Foo = impl Sized; // errors today, no constraining functions
```

```rust
fn weird() -> impl Sized {
    unsafe { std::mem::uninitialized() }
}
// error[E0720]: cannot resolve opaque type
// --> src/lib.rs:1:15
//  |
//1 | fn weird() -> impl Sized {
//  |               ^^^^^^^^^^ recursive opaque type
//2 |     unsafe { std::mem::uninitialized() }
//  |              ------------------------- returning  here with type `impl Sized`
```

* [rust-lang/rust#86732](https://github.com/rust-lang/rust/issues/86732): "min_type_alias_impl_trait doesn't report an error when impl Trait is used in a struct field"
    * we can just close this.

* [rust-lang/rust#86731](https://github.com/rust-lang/rust/issues/86731) "min-type-alias-impl-trait should not accept impl trait in assoc. type binding"
    * this works now, we can close (assuming we have tests).

* [rust-lang/rust#77989](https://github.com/rust-lang/rust/issues/77989): "Type mismatch when using constant with type_alias_impl_trait "
    * we can close this

* [rust-lang/rust#57961](https://github.com/rust-lang/rust/issues/57961): "Defining scope of existential type defined by associated type"
    * the RFC text had notion of defining uses that was pretty narrow in scope. It seems like we want to broad in to include cases like the one listed here.

* [rust-lang/rust#53092](https://github.com/rust-lang/rust/issues/53092): "FullfillmentError ICE with const fn and existential type"
    * opaque type has no bounds, but defining use has a bound on the generic parameter.
    * dies in codegen when someone transmutes into the opaque type.
    * this is reasonable but there is some mishandling with closures; oli has "fix" in https://github.com/rust-lang/rust/pull/96899
    
Next action items, fix everything in https://github.com/rust-lang/rust/issues?q=is%3Aopen+label%3AF-type_alias_impl_trait+assignee%3Aoli-obk (10 issues right now)

Next action item: oli should look at `unconstrained_something_something`
* in cgp module
* https://github.com/rust-lang/rust/blob/83521d2dd09fafc1492ca9ce4664f2f3c59d445e/compiler/rustc_typeck/src/constrained_generic_params.rs#L29
* that function enumerates the parameters
* interesting: identify_constrained_generic_parameters
* https://github.com/rust-lang/rust/blob/83521d2dd09fafc1492ca9ce4664f2f3c59d445e/compiler/rustc_typeck/src/constrained_generic_params.rs#L186
    * when you unify a projection, it may or may not wind up affecting those parameters. so we don't know if they are going to be constrained by opaque types
    * only relevant when/if we can use opaque types in impl headers, but we should panic for now
    * https://github.com/rust-lang/rust/blob/83521d2dd09fafc1492ca9ce4664f2f3c59d445e/compiler/rustc_typeck/src/constrained_generic_params.rs#L62
    * nevermind, already fixed

## 2022-01-19

### Problem 1

This should compile, but doesn't under lazy TAIT

```rust
trait Trait<'a> { }

impl Trait<'b> for &'a u32 { }

fn foo(x: &'x u32) -> impl Trait<'y>
where 'x: 'y
{
    x
}
```

```
error[E0700]: hidden type for `impl Trait` captures lifetime that does not appear in bounds
  --> /home/ubuntu/rust/src/test/ui/impl-trait/region-escape-via-bound-contravariant.rs:20:5
   |
LL | where 'x: 'y
   |       -- hidden type `&'x u32` captures the lifetime `'x` as defined here
LL | {
LL |     x
   |     ^
   |
```

### Problem 2

Doesn't compile with lazy TAIT

```rust
fn contravariant_lub<'a, 'b: 'a, 'c: 'a, 'd: 'b + 'c>(
    x: fn(&'b ()),
    y: fn(&'c ()),
    c: bool,
) -> impl Sized + 'a {
    if c { x } else { y }
}
```

```
error: lifetime may not live long enough
  --> /home/ubuntu/rust/src/test/ui/impl-trait/equal-hidden-lifetimes.rs:41:1
   |
LL |   fn contravariant_lub<'a, 'b: 'a, 'c: 'a, 'd: 'b + 'c>(
   |   ^                        --      -- lifetime `'c` defined here
   |   |                        |
   |  _|                        lifetime `'b` defined here
   | |
LL | |     x: fn(&'b ()),
LL | |     y: fn(&'c ()),
LL | |     c: bool,
LL | | ) -> impl Sized + 'a {
LL | |     if c { x } else { y }
LL | | }
   | |_^ opaque type requires that `'b` must outlive `'c`
   |
   = help: consider adding the following bound: `'b: 'c`

error: lifetime may not live long enough
  --> /home/ubuntu/rust/src/test/ui/impl-trait/equal-hidden-lifetimes.rs:41:1
   |
LL |   fn contravariant_lub<'a, 'b: 'a, 'c: 'a, 'd: 'b + 'c>(
   |   ^                        --      -- lifetime `'c` defined here
   |   |                        |
   |  _|                        lifetime `'b` defined here
   | |
LL | |     x: fn(&'b ()),
LL | |     y: fn(&'c ()),
LL | |     c: bool,
LL | | ) -> impl Sized + 'a {
LL | |     if c { x } else { y }
LL | | }
   | |_^ opaque type requires that `'c` must outlive `'b`
   |
   = help: consider adding the following bound: `'c: 'b`

help: `'b` and `'c` must be the same: replace one with the other
```


## 2021-11-30

```rust
trait T {}
impl T for () {}

fn should_ret_unit() -> impl T {
    panic!()
}
```

```rust
fn should_ret_unit() -> impl T {
    let mut _0: impl T;                  // return place in scope 0 at /home/ubuntu/rust/src/test/ui/never_type/impl_trait_fallback.rs:8:25: 8:31
    let mut _1: !;                       // in scope 0 at /home/ubuntu/rust/library/std/src/panic.rs:18:9: 18:50

    bb0: {
        StorageLive(_1);                 // scope 0 at /home/ubuntu/rust/library/std/src/panic.rs:18:9: 18:50
        begin_panic::<&str>(const "explicit panic") -> bb2; // scope 0 at /home/ubuntu/rust/library/std/src/panic.rs:18:9: 18:50
                                         // mir::Constant
                                         // + span: /home/ubuntu/rust/library/std/src/panic.rs:18:9: 18:32
                                         // + literal: Const { ty: fn(&str) -> ! {std::rt::begin_panic::<&str>}, val: Value(Scalar(<ZST>)) }
                                         // ty::Const
                                         // + ty: &str
                                         // + val: Value(Slice { data: Allocation { bytes: [101, 120, 112, 108, 105, 99, 105, 116, 32, 112, 97, 110, 105, 99], relocations: Relocations(SortedMap { data: [] }), init_mask: InitMask { blocks: [16383], len: Size { raw: 14 } }, align: Align { pow2: 0 }, mutability: Not, extra: () }, start: 0, end: 14 })
                                         // mir::Constant
                                         // + span: /home/ubuntu/rust/library/std/src/panic.rs:18:33: 18:49
                                         // + literal: Const { ty: &str, val: Value(Slice { data: Allocation { bytes: [101, 120, 112, 108, 105, 99, 105, 116, 32, 112, 97, 110, 105, 99], relocations: Relocations(SortedMap { data: [] }), init_mask: InitMask { blocks: [16383], len: Size { raw: 14 } }, align: Align { pow2: 0 }, mutability: Not, extra: () }, start: 0, end: 14 }) }
    }

    bb1: {
        StorageDead(_1);                 // scope 0 at /home/ubuntu/rust/library/std/src/panic.rs:18:49: 18:50
        return;                          // scope 0 at /home/ubuntu/rust/src/test/ui/never_type/impl_trait_fallback.rs:10:2: 10:2
    }

    bb2 (cleanup): {
        resume;                          // scope 0 at /home/ubuntu/rust/src/test/ui/never_type/impl_trait_fallback.rs:8:1: 10:2
    }
}

```


```rust
trait T {}
impl T for () {}

type Foo = impl T;

fn omg() -> Foo {
    panic!()
}

fn foo() -> Foo {
    22
}
```

```rust
trait T {}
impl T for () {}

type Foo = impl T;

fn omg() -> Foo {
    panic!()
}

fn passthrough() -> Foo {
    omg()
}

fn foo() -> Foo {
    22
}
```


```rust
#![feature(never_type_fallback)]
trait T {}
impl T for () {}
impl T for u32 {}

fn should_ret_unit() -> impl T {
    panic!()
}
```

| Impl 1 | Impl 2 | Stable | `#![feature(never_type_fallback)]` |
| ------ | ------ | ------ | ------ |
| `()` | `u32` | result is `()`, ok | result is `!`, error |
| `i32` | `u32` | result is `()`, error | result is `!`, error |
| `()` | `!` | result is `()`, ok | result is `!`, ok |
| `!` | N/A | result is `()`, error | result is `!`, ok |
| `u32` | N/A | result is `()`, error | result is `!`, error |

```rust
#![feature(never_type_fallback)]
trait T {}
impl T for () {}
impl T for u32 {}

fn should_ret_unit() -> impl T {
    loop {
        
    }
}

fn fun() -> impl T {
    fun()
}
```

if we say:

* use `!` type (or `()`) if there are no constraints
    * at least in the case of RPIT
* then this code will work:

```rust
#![feature(never_type_fallback)]
#![feature(never_type)]

trait T {}

fn should_ret_unit1() -> impl T {
    should_ret_unit2()
}

fn should_ret_unit2() -> impl T {
    should_ret_unit1()
}

fn main() { }
```

but it did not before.


```rust
#![feature(never_type)]

fn fun() -> ! {
    foo()
}

fn foo() -> ! {
    fun()
}
```

### Example 2

```rust

#![feature(associated_type_bounds)]

use std::ops::Add;

trait Tr1 { type As1; fn mk(&self) -> Self::As1; }
trait Tr2<'a> { fn tr2(self) -> &'a Self; }

fn assert_copy<T: Copy>(x: T) { let _x = x; let _x = x; }
fn assert_static<T: 'static>(_: T) {}
fn assert_forall_tr2<T: for<'a> Tr2<'a>>(_: T) {}

struct S1;
#[derive(Copy, Clone)]
struct S2;
impl Tr1 for S1 { type As1 = S2; fn mk(&self) -> Self::As1 { S2 } }

fn def_et1() -> Box<dyn Tr1<As1: Copy>> {
    let x /* : Box<dyn Tr1<As1: Copy>> */ = Box::new(S1);
    x
}
pub fn use_et1() { assert_copy(def_et1().mk()); }

fn def_et2() -> Box<dyn Tr1<As1: Send + 'static>> {
    let x /* : Box<dyn Tr1<As1: Send + 'static>> */ = Box::new(S1);
    x
}
pub fn use_et2() { assert_static(def_et2().mk()); }

fn def_et3() -> Box<dyn Tr1<As1: Clone + Iterator<Item: Add<u8, Output: Into<u8>>>>> {
    struct A;
    impl Tr1 for A {
        type As1 = core::ops::Range<u8>;
        fn mk(&self) -> Self::As1 { 0..10 }
    }
    let x /* : Box<dyn Tr1<As1: Clone + Iterator<Item: Add<u8, Output: Into<u8>>>>> */
        = Box::new(A);
    x
}
pub fn use_et3() {
    let _0 = def_et3().mk().clone();
    let mut s = 0u8;
    for _1 in _0 {
        let _2 = _1 + 1u8;
        s += _2.into();
    }
    assert_eq!(s, (0..10).map(|x| x + 1).sum());
}

fn def_et4() -> Box<dyn Tr1<As1: for<'a> Tr2<'a>>> {
    #[derive(Copy, Clone)]
    struct A;
    impl Tr1 for A {
        type As1 = A;
        fn mk(&self) -> A { A }
    }
    impl<'a> Tr2<'a> for A {
        fn tr2(self) -> &'a Self { &A }
    }
    let x /* : Box<dyn Tr1<As1: for<'a> Tr2<'a>>> */ = Box::new(A);
    x
}
pub fn use_et4() { assert_forall_tr2(def_et4().mk()); }

fn main() {
    let _ = use_et1();
    let _ = use_et2();
    let _ = use_et3();
    let _ = use_et4();
}

```

## 2021-08-30

* Discussion on Zulip today
* Santiago finished test PRs https://github.com/rust-lang/rust/issues/86727
    * [this table summarizes the details](https://hackmd.io/y1HwFSBUQS2ZKrSFuQ0Dnw?view)

## 2021-08-23

### The test table

[the table](https://hackmd.io/y1HwFSBUQS2ZKrSFuQ0Dnw?view)

**Use outside of a “defining use”**

```rust
type Foo = impl Debug;

fn test1() -> u32 {
    let x: Foo = 22_u32; // ACCEPTED
    x // based on the RFCs, legal
}

fn test1b() -> u32 { // ACCEPTED
    let x: Foo = 22_u32;
    let y: Foo = x;
    same_type((x, y)); // OK
}

// Equivalent-ish?
fn test2() -> u32 { // FEATURE GATED
    let x: impl Debug = 22_u32;
    x // ERROR: we only know x: Debug, we don't know x = u32
}

fn test2b() -> u32 { // FEATURE GATED
    let x: impl Debug = 22_u32;
    let y: impl Debug = x;
    same_type((x, y)); // ERROR
}

fn same_type<T>(x: (T, T)) { }
```

```rust
type Foo = impl Debug;

// Allowed, and what do we expect it to do?
fn foo(mut x: Foo) {
   x = 22_u32;
}

// Also allowed
fn foo(mut x: Foo) {
    // no constraint on x
}
    
// Not allowed (error) under current setup
fn foo(x: Foo) {
    println!("{:?}", x);
}
```

```rust
type Foo = impl Debug;

// OK
struct SomeStruct {
    f: Foo // impl Debug syntax in this case forbidden
}

fn foo() {
    let f = SomeStruct { f: 22_u32 };
}
```

```rust
type Foo = impl Debug;

// Errors today, could eventually work
static FOO: Foo = 22_u32;

// Feature gated
static FOOb: impl Debug;
```

### Auto trait leakage

```rust
mod m {
    type Foo = impl Debug;
    
    pub fn foo() -> Foo { 22_u32 }
}

fn is_send<T: Send>(_: T) { }

fn main() {
    is_send(m::foo()); // OK
}
```

```rust
mod m {
    type Foo = impl Debug;
    
    pub fn foo() -> Foo { Rc::new(22_u32) }
}

fn is_send<T: Send>(_: T) { }

fn main() {
    is_send(m::foo()); // error
}
```


```rust
mod m {
    type Foo = impl Debug;
    
    pub fn foo() -> Foo { 22_u32 }

    pub fn bar() {
        is_send(foo()); // Today: error
    }
    
    fn is_send<T: Send>(_: T) { }
}
```

### Inference cycle

```rust
mod m {
    type Foo = impl Debug;
    
    // Cycle: probably an error today, but it'd
    // be nice if it eventually worked
    
    pub fn foo() -> Foo { 
        is_send(bar())
    }

    pub fn bar() {
        is_send(foo()); // Today: error
    }
    
    fn baz() {
        let f: Foo = 22_u32;
    }
    
    fn is_send<T: Send>(_: T) { }

}
```


### weird return types

```rust
type Foo = impl Debug;

fn f() -> impl Future<Output = Foo> {
    async move { 22_u32 }
}
```

### Oli's turn

* Last time we dug into a problem involved sized obligations on opaque types
    * on Friday, we found that when we try to match them against where clauses in scope, the equality "just works"
    * and the trait matching code has some logic that it understand unbound type inference variables but it doesn't trigger
    * Niko suggested generating inference variables for opaque types to let the existing logic work
        * Problem: What we need to do is to update the code `resolve_if_possible` to replace opaque types and not just inference variables
        * In case that an opaque type is in-scope but not known, we would generate an inference variable
            * Problem: to do that, we will have to generate obligations
            * Which alters the return type for this function
            * OMG HELP ME
        * Alternative:
            * everywhere we check inference variables, consider opaque types that are in scope as well
            * ok, niko approves
    * 

### two out-of-scope opaque types referencing one another

Good

```rust
mod a {
    pub type A: impl Debug;
    pub fn get_a() -> A { 22_u32 }
}

mod b {
    pub type B: impl Debug;
    pub fn get_b() -> B {
        crate::a::get_a();
    }
}
```

Not so good

```rust
mod a {
    pub type A: impl Debug;
    pub fn get_a() -> A {
        crate::b::get_b();
    }
}

mod b {
    pub type B: impl Debug;
    pub fn get_b() -> B {
        crate::a::get_a();
    }
}

fn main() {
   a::get_a();
}
```


#### two opaque types interacting

```rust
type Foo = impl Debug;
type Bar = impl Debug;

fn cheat(f: Foo) {
    let b: Bar = f;
    let c: u32 = b;
}

fn main() { }
```


```rust
type Foo0 = impl Debug;
type Foo1 = impl Debug;
type Foo2 = impl Debug;

fn cheat(f: Foo0) {
    let b: Foo1 = f; // Foo1 maps to Foo0
    let c: Foo2 = ; // Foo2 maps to Foo0
    
    let c: u32 = b;
}

fn main() { }
```

#### test table

| Test | 2021-08-23 | 2021-08-30 (after oli's branch lands) |
| --- | --- | --- |
| blahblahblah.rs | :red_circle: | :ballot_box_with_check: |
| blahblahblah.rs | :ballot_box_with_check: | :ballot_box_with_check: |
| **out of scope** |
| blahblahblah.rs | :red_circle: | :red_circle: |


In test file:

```
// feature: foo(subcategory)
// fixme: #123  -- means that the stderr files represent buggy behavior
```

x.py test --report foo 

## 2021-08-20

* multiple problems
    * inference context has the ability to probe, unroll
    * lacking information about an opaque type
        * whenever we try to equate a type with an opaque type
        * if it's a defining use, we take that value as its value
            * generate obligations <-- note to follow up on: how do we do that
            * passed back through this InferOk
        * in a probe, this naturally succeeds
        * we are rolling back the state of the opaque type


## 2021-08-16

* oli-obk's `lazy_tait2` branch is almost working but hitting some kind of error
    * includes various refactorings completed that allow registering obligations at the times needed
* `min_type_alias_impl_trait` is removed
* test table is complete, needs review
* 

## 2021-07-19

### Status checkin

* oli:
    * Merged some of oli's PRs, no wronger than what we had before (woohoo)
    * Currently looking into the "lazy opaque types" we described
    * Type check step is pretty much useless except for preventing cycles
    * MIR borrow check basically has to recompute all the information anyway
    * https://github.com/rust-lang/rust/blob/8df945c4717ffaf923b57bf30c473df6fc98bc85/compiler/rustc_typeck/src/collect/type_of.rs#L531-L542
        * just looks at *what types are constrained*
    * Breaking the connection between MIR borrow check and type check is step 1
* spastorino:
    * PR https://github.com/rust-lang/rust/pull/87141 is r+d
    * Test case coverage?
* 

## 2021-07-16

* [oh dear god](https://github.com/rust-lang/rust/blob/8240e7aa101815e2009c7d03b33dd2566d843e73/compiler/rustc_typeck/src/check/mod.rs#L355)
* why we needed this
    * if you have inference variables in the typeck results, bad things happen
* lazy strategy
    * add to the inference context a `Option<DefId>` that is the "impl trait scope"
        * opaque types inside this scope are being inferred
        * opaque types outside this scope are opaque
        * None means all opaque types are opaque
* the list
    * typeck
    * borrowck
    * MIR borrowck
    * MIR typeck
* how it works today
    * typeck infers an "erased" version of the hidden type
        * (probably some old regionck stuff happens here too)
    * MIR typeck
        * reads the erased version from typeck
            * instantiates the regions with variables
        * equates the erased version with the constraints from the fn
            * thus constraining the region variables
    * MIR borrowck
        * solves the constraints, yielding values for the region variables
        * are then mapped to the formal type parameters (hopefully)
    * the `type_of` query, when trying to get the revealed type,
        * executes MIR borrowck
        * reads the results and returns them
* apparent caveat because we can't have nice things
    * this `fixup_` thing sometimes resolves them to the declarations generics lifetimes (but in some incorrect way)
* drilling into the first bullet point
    * when you start to type check the function
        * we find the `impl Trait` types that appear in the signature
            * which are within the defining scope (`may_define_opaque_type`)
        * replace them with inference variables
            * we keep a map, keyed by def-id+substs, that goes back to the variable and some other crap
        * we add constraints that the inference variable must implement the relevant traits
    * we run type-check
    * we look at the final value of the inference variable
        * assuming it is fully known we write it back into the typeck table
        * after erasing regions
        * and apparently also randomly changing to incorrect values
* what Niko wants to change
    * remove the instantiate stuff as a "pre-step"
    * do it when we encounter the type during unification
        * the tables move into fields of the inference context
* inference context has that method `resolve_variables_if_possible`
    * here and there, there is probably some code that doesn't invoke this function which will have to
* probably the most reliable way to approach this, but churn-y way
    * is to have a `ty::PlaceholderOpaqueType` that you map to
    * so that `ty::OpaqueType`

```rust=
type Foo = impl Clone;

fn foo(x: Foo) {
    x.clone()
}
```

```rust=
type Foo = impl Clone;

struct Bar {
    f: Foo
}

fn my_func() {
    let b = Bar { f: 22_u32 };
} // constraint Foo = u32
```

```rust=
type Foo = impl Clone;

fn foo() {
    let f: Foo = 22_u32;
    // I do get to assume `f = u32`
}

fn bar() {
    let f: impl Clone = 22_u32;
    // I don't get to assume `f = u32`, just that `f: Clone`
}


```


## 2021-07-14

```rust
type TAIT1 = impl Debug;

trait TheTrait {
    type TheType;
}

impl TheTrait for () {
    type TheType = TAIT1;
}

fn foo(mut x: <() as TheTrait>::TheType) {
    x = u32;
}
```

MCP:



## 2021-07-12

* Is something missing? Oli to do triage.
    * Go over the things tagged as F-type-alias-impl-trait
    * Rubric:
        * (a) Does this reproduce if you only use `min_type_alias_impl_trait`?
            * If no: no problem
        * (b) If so, should it? 
            * If no: file an issue that this case should be excluded by `min_type_alias_impl_trait`
        * (c) Else, tag the issue as `F-min_type_alias_impl_trait`
* Oli:
    * repeated TAIT doesn't detect lifetime conflict #86465 
        * https://github.com/rust-lang/rust/pull/86410
        * PR fixes the bug but introduces a new bug
        * Doesn't bootstrap *in stage2* but we don't know why
            * Seems to be specific to nested impl trait
        * `cargo test +rust-0-stage1 src/test/ui`
            * existing tests seem to catch the problem
    * Defining scope of existential type defined by associated type https://github.com/rust-lang/rust/pull/57961 
        * Out of scope, but we should adjust the example
            * 

```rust=
trait Foo {
    type Output1: Iterator<Item = Self::Output2>;
    type Output2;
}

impl Foo for () {
    type Output1 = vec::IntoIter<u32>;
    type Output2 = impl Sized;
}
```

```rust=
trait Identity {
    type Eq;
}

impl<T> Identity for T {
    type Eq = T;
}

trait Foo {
    type Output1: Identity<Eq = Self::Output2>;
    type Output2;
}

impl Foo for () {
    type Output1 = u32;
    type Output2 = impl Sized;
}
```

Options:

* Tod

```rust=
trait Foo {
    type Output1: PartialEq<Self::Output2>;
    type Output2;
}

impl Foo for () {
    type Output1 = vec::IntoIter<u32>;
    type Output2 = impl Sized;
}
```



```rust=
type Foo = impl Trait;
type Bar = (Foo, u32);

fn foo() -> Bar { }
```

```rust=
type Bar = (impl Trait, u32);

fn foo() -> Bar { }

```

### Outcome from this discussion

* Existing restrictions were meant to enforce this invariant:
    * Invariant Niko wanted:
        * Either it's a universal use (`fn foo(x: impl Trait)`)
        * Or it's a defining use (`fn foo() -> impl Trait`)
    * That rules out:
        * `type Foo = impl Trait; fn bar(x: Foo)` -- not a defining use today
        * `fn bar(x: <() as Foo>::Output2)`
    * Reason why:
        * Current code handles the above cases as "opaque" but RFC suggests that we should be able to infer the value of the TAIT from the above examples
* You can bypass the `min_type_alias_impl_trait` restrictions by using normalization
    * [example](https://play.rust-lang.org/?version=nightly&mode=debug&edition=2018&gist=e84a6662ebee05345e8d858d920c41de)
    * plausibly: we should alter these restrictions so that we are able to infer the value of the TAIT
        * but we would not handle method dispatch like `x.method()` (who cares)
* Actually the restrictions just don't exist:
    * [example](https://play.rust-lang.org/?version=nightly&mode=debug&edition=2018&gist=de0dd19691adc995610eb4263cd035dc)
* When you use a `impl Trait` in an impl, it appears as opaque
    * [example](https://play.rust-lang.org/?version=nightly&mode=debug&edition=2018&gist=949f5fa73e29b506bfba0a7fd68e0436)

## Result of today's meeting

Make *everything* be a defining use that typeck figures to be a defining use via https://doc.rust-lang.org/nightly/nightly-rustc/rustc_middle/ty/struct.TypeckResults.html#structfield.concrete_opaque_types
