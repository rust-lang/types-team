# Implied bounds and well-formedness checking

[Zulip topic for discussion](https://rust-lang.zulipchat.com/#narrow/stream/326132-t-types.2Fmeetings/topic/meeting.202022-07-08)

## Goal

To reconcile compiler's notion of implied bounds, in-scope types, and WF checking with something in a-mir-formality that can live with and that is simple and easy to understand.

Also, to figure out how to integrate implied bounds into HRTB so as to address GAT usability concerns.

## Terminology

In writing this document, I realized that I have interchangeably used the terms implied bounds for multiple things. I'm going to avoid the term implied bound and introduce some distinctions:

* *Elaborated bounds* refers to cases where we figure out, based on one where clause, other things that must be true. For example, supertraits, where `T: Ord` implies that `T: Eq` also holds. I'll call that an elaborated bound.
* *Default bounds* refers to cases where we add where-clauses of various kinds. We do this for two reasons:
    * *In-scope types* -- look at types that appear in a declaration and use that to imply some where-clauses that must hold. For example, `fn foo<'a, T>(x: &'a T)` considers `&'a T` to be an in-scope type, and hence can conclude that `T: 'a` must hold.
    * *Higher-ranked* -- to make higher-ranked bounds make sense.

Implied bounds are then the union of elaboarted + defaulted. In both cases, they refer to facts that we are able to conclude that were not explicitly written by the user and which were not synthesized using impls.

## Integrating in-scope types into a-mir-formality and Rust

The main goal for in-scope types has been to convert them into implicit where clauses that get added very early on in the process and which can be ignored from that point forward. I've been modeling this in a-mir-formality. 

To get a sense for what I mean, consider the following list of mir-formality layers and their rough rustc equivalents (note that this doesn't quite correspond to what's in the codebase; in the act of writing this doc I decided it made sense to split the existing "mir" layer into check/body):

| Layer | rustc | Defines | Requires from upper layers via hook |
| --- | --- | --- | --- |
| rust | AST | user-facing syntax for decls/bodies, how that lowers into internal syntax | |
| check | HIR | overall check of the Rust program |  |
| body | MIR type checker | grammar of fn bodies, type and borrow checker |  |
| decl | HIR-level queries, traits | grammar of Rust declarations, program clauses derived from Rust syntax like traits/impls | 
| ty | ty module, relate, infer | grammar of Rust types, definition of relations | generic types for a given ADT |
| logic | traits  | what is provable | set of program clauses, definition of relations, semantics of a predicate |

In [my branch](https://github.com/nikomatsakis/a-mir-formality/tree/biformulas-everywhere), in-scope types are introduced as part of the **lowering in the rust layer** (the equivalent of AST->HIR lowering in rustc). 

As an example, consider a function like

```rust
fn foo<'a, T>(x: &'a T)
where
    T: Ord
```

Or, in the grammar of a-mir-formality's rust layer:

```
(fn foo[(lifetime a) (type T)]((& a T)) -> ()
    where [(T : Ord[])]
    {trusted-fn-body})
```

When this function gets lowered into the "decl layer", we wind up with the following where clauses:

* `(is-implemented (Ord [T]))` -- derived from `(T : Ord[])` ([code](https://github.com/nikomatsakis/a-mir-formality/blob/8a83f2c55425d220d89d78b6e1039dfdcfed6bed/src/rust/lower-to-decl/fn-decl.rkt#L15))
* `(well-formed (type (& a T)))`[^user-ty] -- in-scope type, automatically inserted ([code](https://github.com/nikomatsakis/a-mir-formality/blob/8a83f2c55425d220d89d78b6e1039dfdcfed6bed/src/rust/lower-to-decl/fn-decl.rkt#L16-L17))

[^user-ty]: The type `(& a T)` here is not the actual a-mir-formality syntax. That's the "user type", and to be correct, we ought to write `(user-ty (& a T))`, which invokes a function to convert to the internal representation. The internal representation is `(rigid-ty (ref ()) [a T])`, but you can see why I didn't want to write that.

Here, for example, is the current [code for lowering a function](https://github.com/nikomatsakis/a-mir-formality/blob/8a83f2c55425d220d89d78b6e1039dfdcfed6bed/src/rust/lower-to-decl/fn-decl.rkt#L10-L22). The `👈🥱` line marks the point where the user-given where-clauses are converted. The `👈🚨` lines mark the where clauses for in-scope types.

```
(define-metafunction formality-rust
  lower-to-decl/FnDecl : Rust/FnDecl -> FnDecl

  [(lower-to-decl/FnDecl (fn FnId KindedVarIds (Rust/Ty_arg ...) -> Rust/Ty_ret 
                             where [Rust/WhereClause ...] FnBody))
   (fn FnId KindedVarIds ((user-ty Rust/Ty_arg) ...) -> (user-ty Rust/Ty_ret)
       where [(lower-to-decl/WhereClause Rust/WhereClause) ...  ; 👈🥱
              (well-formed (type (user-ty Rust/Ty_arg))) ...    ; 👈🚨
              (well-formed (type (user-ty Rust/Ty_ret)))        ; 👈🚨
              ]
       FnBody)
   ]
  )
```

## Elaborating in-scope types

In the previous section, we talked about how an in-scope type resulted in a where-clause `(well-formed (type (& a T))` being added to the function. But the *effect* of this is that the fn is supposed to be able to conclude that `T: 'a` (or, in mir-formality terms, `T -outlives- a`)...how does that work?

This is where **elaboration** comes in. We have a concept of elaboration which takes the set of hypotheses (where-clauses assumed to be true) and converts them into a richer set. This is done based on *invariants*, and one such invariant is

* `(well-formed (type (& a T)))` implies `(T -outlives- a)`

In other words: if you know that `&'a T` is well-formed, you can assume that `T: 'a` must also be true. We introduce other invariants based on struct declarations and where clauses. So for example `struct Foo<'a, 'b, T> where T: 'a + 'b` introduces two invariants

* `(well-formed (type (Foo < a b T >)))` implies `(T -outlives- a)`
* `(well-formed (type (Foo < a b T >)))` implies `(T -outlives- b)`

### Other kinds of elaboration

We haven't fully implemented the elaboration of outlives types in a-mir-formality yet, that is [a-mir-formality/#63](https://github.com/nikomatsakis/a-mir-formality/issues/63). In rustc, if you know that `Vec<T>: 'a`, we conclude that `T: 'a`. This is based on the (hard-coded) outlives rules for when `Vec<?T>: 'a` (iff `?T: 'a`). This kind of reasoning is very important in traits and impls. Consider this case:

```rust
trait Get {
    type T;
    fn get(&'g self) -> &'g Self::T;
}

impl<'a, T> Get for &'a T {
    type T = T;
    
    fn get<'g>(&'g self) -> &'g T {
        &*self
    }
}
```

This impl looks reasonable, right? But if you look closely at the definition of `get`, you'll see there's something subtle going on:

* `&*self` yields a value of type `&'a T`
* but the return type is `&'g T`

How do we know that `'a: 'g`, and hence it is legal to upcast? For that matter, how do we know that `T: 'g`?

If you consider the trait/impl pair with in-scope types written in full, it becomes more clear:

```rust
trait Get {
    type T;
    fn get(&'g self) -> &'g Self::T
    where
        well_formed(&'g Self);
}

impl<'a, T> Get for &'a T {
    type T = T;
    
    fn get<'g>(&'g self) -> &'g T
    where
        well_formed(&'g &'a T)
    {
        &*self
    }
}
```

The impl's where-clause follows from the trait where-clause, and because `&'g &'a T` is well-formed, we can conclude that `T: 'a: 'g`. But deriving all those facts relies on a combination of elaboration and invariants.

## How does this notion of in-scope types compare with rustc's behavior?

When it comes to in-scope types on functions and impls, the mir-formality version roughly matches rustc's behavior, except that it closes soundness holes like [#25860](https://github.com/rust-lang/rust/issues/25860). 

Part of why those soundness holes arise is because rustc integrates in-scope types in a kind of ad-hoc, complex way around the codebase. We have things that the callee is assuming to be true (`T: 'a`) but they are not listed explicitly as facts the caller must prove and -- indeed -- the caller does not always wind up proving them.

### Can we just do this in rustc then and fix the bug?

Not now, but eventually yes! The reason that rustc is setup the way it is is to deal with the early- vs late-bound distinction. Consider this function:

```rust
fn foo<T>(x: &T) {
}
```

The type of `foo` is `for<'a> fn(&'a ?T)` -- i.e., `T` is converted into an inference variable when you reference it, but `'a` remains *late-bound*. If we were to add a where-clause during AST->HIR conversion, however...


```rust
fn foo<'a, T>(x: &'a T) 
where
    T: 'a // or `well_formed(&'a T)`, if we had that, etc
{
}
```

...then `'a` would become *early-bound*, meaning that the type of `foo` would be `fn(&'?a ?T)`, where both `'a` and `T` become inference variables. This is required because we have to register the where-clause `?T: '?a` to be proven at some point.

If we had richer types, though, the type of `foo` could be

```
for<'a, T> where (T: 'a) fn(&'a T)
```

...and this is precisely what a-mir-formality supports.

*That said*, we might be able to do *something* in this direction, because of the fndecl vs fnpointer distinction (which I was ignoring here). Worth thinking about, perhaps.

## Well-formedness checking

OK, so we have a notion of in-scope types -- how does that fit with **well-formedness checking**? This term is again a kind of "catch-all" that we use for various things in rustc. I'm going to be more specific:

* A type is *well-formed* if it (a) all of its constituent types are well-formed and (b) it meets all of the where clauses declared on the relevant type (or built-in, in the case of things like `&T`).
    * So `Vec<T>` is well-formed if (a) `T` is well-formed and (b) `T: Sized` is true (since `Vec<T>` has `where T: Sized` and nothing else).
* A where-clause is *well-formed* if (a) all of its constituent terms (types or where-clauses) are well-formed and (b) it meets all of the where clauses declared on the relevant trait (if any).
    * So `T: Ord` is well-formed if (a) `T` is well-formed and (b) `T: Eq` is true, since `trait Ord` has `where Self: Eq` (i.e., a supertrait).
    * Similarly `Foo: 'a` is well-formed if (a) `Foo` is well-formed and `'a` is well-formed (there are no where-clauses to consider, so (b) doesn't apply).
* A **declaration** is well-formed if (a) all of its constituent terms are well-formed. There are no where-clauses to meet[^check-vs-wf].
    * So `impl<T> Ord for Vec<T> where T: Ord` is legal if...
        * the type `Vec<T>` is WF
        * the where clause `T: Ord` is WF
        * the where clause `Vec<T>: Ord` is WF (it's a bit of a stretch, but we'll say the trait-ref being implemented is a constituent term)

[^check-vs-wf]: You could say that the "where clauses declared elsewhere that must be met" for a declaration are things like: an impl must ensure that it meets the signatures declared on the trait, etc. Or similarly, that a struct's fields must be well-formed. That's what rustc does, I think, when we do these checks in wf checking. I've changed it because (as I discuss below) I want to maintain the invariant in mir-formality that WF checking is NOT required for soundness; the rules exist to meet user expectations and because of semver considerations. At present, this split is not as clear as I would like in mir-formality, I'm trying to establish it.

### Elaboration and WF checking

When we check well-formedness for a declaration, we do so in an environment that assumes that all the where-clauses on the declaration hold (and their elaborations). Consider the rule that says, for `T: Ord` to be WF, `T: Eq` must hold -- we don't require explicitly writing `T: Eq`, why is that? It's because it follows from `T: Ord` by elaboration. However, because our elaboration only elaborates supertrait where-clauses, some where-clauses declared on the trait *do not follow* implicitly and must be declared explicitly:

```rust
trait BothOrd<T: Ord>: Ord {}

fn foo<A, B>()
where
    A: BothOrd<B>
{}
```

This function is [not well-formed](https://play.rust-lang.org/?version=stable&mode=debug&edition=2021&gist=7a37c6f128fdc7b3b1077a815419ae6f) because it lacks `B: Ord`. This is because `A: BothOrd<B>` is well-formed if `A: Ord` and `B: Ord`. The first follows by elaboration from `A: BothOrd<B>`, but the second does not.

### In mir-formality, WF checking is a "nice to have"

The way that mir-formality is setup, WF checking is NOT required for soundness. Loosely speaking, the idea is this: declarations are allowed to assume that their constituent terms are well-formed. The caller is responsible for proving it.

One of the ways this plays out is: to prove `(is-implemented (SomeTrait [SomeType]))`, in mir-formality, we must prove both that

* `(has-impl (SomeTrait [SomeType]))` -- i.e., there is an impl somewhere
* and that the `(SomeTrait [SomeType])` is well-formed, meaning that (a) `SomeType` is WF and (b) the where-clauses on `SomeTrait` hold.

In rusc, this works differently. We only prove the first part, and we assume (thanks to WF checking etc) that the secondary stuff holds. But this doesn't work when you extend to coinduction. In that case, it's possible to have self-supporting impls (at least unless we impose some conditions on the kinds of where-clauses you can have in impls, and I've not found a formulation of those conditions that accepts the kind of code we currently accept).

So, if WF checking is not required, why do we do it? To be honest, I'm not totally sure we should! I think it might be better to make WF checking a kind of lint (i.e., detecting cases where the caller couldn't possibly prove things are WF and warning about it, but otherwise accepting what was written). But I'm trying to model rustc roughly as it is so that we can then describe proposed changes in terms of the model.

### Defining well-formedness

So, in (my branch of) mir-formality, I've defined some functions that create the `Goal` required to prove a given term (type, where-clause) is well-formed:

* [`well-formed-goal-for-ty`](https://github.com/nikomatsakis/a-mir-formality/blob/8a83f2c55425d220d89d78b6e1039dfdcfed6bed/src/decl/well-formed/parameter.rkt#L17) takes a type and gives back the goal to prove it well-formed
* [`well-formed-goal-for-lt`](https://github.com/nikomatsakis/a-mir-formality/blob/8a83f2c55425d220d89d78b6e1039dfdcfed6bed/src/decl/well-formed/parameter.rkt#L31) does the same for lifetimes (not much to do there...)
* [`well-formed-goal-for-biformula`](https://github.com/nikomatsakis/a-mir-formality/blob/8a83f2c55425d220d89d78b6e1039dfdcfed6bed/src/decl/well-formed/biformula.rkt#L18)[^biformula] does the same for where-clauses

[^biformula]: Biformula? WTF is that? This "mir-formality" for a lowered "where-clause", kind of a `ty::Predicate` in rustc. The name biformula is meant to suggest it is a logical formula that can serve *either* as a goal (something you have to prove) or a clause/hypothesis (something you can assume is true). An earlier name was `Goal∧Clause` but that got annoying to read and pluralize. :)

There is also a predicate `(well-formed (type Ty))`. Proving that predicate requires proving `(well-formed-goal-for-ty Ty)`. This predicate is in fact "built-in" by the decl layer, meaning that we define it with some custom code; this is needed (a) to account for the infinite variety of types (e.g., tuples of arbitrary arity) in ways that we don't currently permit users to quantify (i.e., you can't write an impl that covers tuples of arbitrary arity), but also (b) to gently guide the trait solver away from some hopeless corners (i.e., proving that `(well-formed (type ?T))` is true by enumerating the (infinite) set of well-formed types `?T`).

## Extending in-scope types to higher-ranked things

So far, everything we said matches rustc's behavior (more or less). Now let's talk about higher-ranked types and trait bounds. Rustc's behavior around these is kind of inconsistent and not very useful. I spent a while trying to [rationialize and model it](https://hackmd.io/0tbjoWROTGC3uqGFF_iVHw), with some success,  but ultimately let's talk about how to make it useful.

I'm going to start by focusing on *higher-ranked trait bounds* and come back to higher-ranked types.

### Problem: `for<'a>` shouldn't really mean for **ALL** `'a`

Right now, when you write `for<'a> Predicate`, it means that `Predicate`
 is true **for all** `'a`. That turns out to not be an especially useful definition. Almost always what we want is more like **for any suitable `'a`**. Here is a rather artificial example to show you what I mean. Consider a trait `IntoData`, that lets you convert some type into a slice with lifetime `'d`, so long as `'d` is "suitably short"[^assoc]:
 
[^assoc]: Side note that this is probably not the way to model this problem, you'd probably prefer an associated type for the slice, or perhaps an associated lifetime, if we had those. But never mind.
 
```rust
trait IntoData<'d>
where
    Self: 'd,
{
    fn into_data(self) -> &'d [u32];
}

impl<'d> IntoData<'d> for &'d [u32] { // example impl
    fn into_data(self) -> &'d [u32] {
        self
    }
}
```

Now we might like to define a function that takes some `T` that implements `MakeRef`:

```rust
fn foo<T>(t: T)
where
    T: for<'short> IntoData<'short>
{
    let data: &[u32] = t.into_data();
    process(data);
}
```

But what happens when I call foo? ([playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2021&gist=8f0ed5d96f022fba2e0c367edf119a45))

```
error: implementation of `IntoData` is not general enough
  --> src/main.rs:27:5
   |
27 |     foo(s);
   |     ^^^ implementation of `IntoData` is not general enough
   |
   = note: `IntoData<'0>` would have to be implemented for the type `&[u32]`, for any lifetime `'0`...
   = note: ...but `IntoData<'1>` is actually implemented for the type `&'1 [u32]`, for some specific lifetime `'1`
```

Huh, well *that's* annoying. The error makes sense though -- we can't really say that `T: IntoData<'short>` for **any** lifetime `'short`, only for lifetimes that are 'suitably short' (in particular, those where `T: 'short`). 

Of course, this example is kind of artificial and silly. Part of the reason for that is that in-scope types on impls often allow us to circumvent these problems. For example, I could write `IntoData` somewhat differently:

```rust
trait GetData {
    fn data(&self) -> &[u32]
}
```

...and it would work for that particular example. Of course, the reason it works is because of in-scope types, as we saw earlier. And, of course, this is not the same interface: I can't use this interface to convert a `T` into a `&[u32]` that I can return from the function.

This problem arises a lot with GATs. Consider this:

```rust
trait LendingIterator {
    type Item<'a> where Self: 'a;
}

fn foo<T>()
    for<'a> T: LendingIterator<Item<'a> = &'a u32>
{
    
}
```

Now imagine calling `foo` for some specific type `MyLendingIter<'b>`. To prove that `MyLendingIter<'b>: LendingIterator<Item<'a> = &'a u32>`, we have to show that the where-clause `Self: 'a` holds, which means `MyLendingIter<'b>: 'a` -- and we have to show that **for any** `'a`. But of course that's not true, so we will get a similar error.

### Solution: add defaulted bounds

How can we solve the above? We need some default bounds. Roughly speaking, my idea is to say that, when we convert `for<P> WhereClause` to a biformula, instead of doing [this](https://github.com/nikomatsakis/a-mir-formality/blob/8a83f2c55425d220d89d78b6e1039dfdcfed6bed/src/rust/lower-to-decl/where-clause.rkt#L12-L13):

```
  [(lower-to-decl/WhereClause (for KindedVarIds Rust/WhereClause))
   (∀ KindedVarIds (lower-to-decl/WhereClause Rust/WhereClause))]
```

...we will do something like this[^gerk]...

[^gerk]: For various annoying "needs refactoring"-type reasons, you couldn't copy and paste this into mir-formality as is. We have to ensure that the `well-formed-goal-for-biformula` returns a biformula, and not an arbitrary goal, for example (I think it does), and that we don't include `&&` connectives (not sure about that), or else that we normalize them, and a few other annoying things.

```
  [(lower-to-decl/WhereClause (for KindedVarIds Rust/WhereClause))
   (∀ KindedVarIds 
     (implies [Biformula_wf] Biformula))
   )
   (where/error Biformula (lower-to-decl/WhereClause Rust/WhereClause))]
   (where/error Biformula_wf (well-formed-goal-for-biformula Biformula))
```

In other words, `for<'a> WC` where `WC` is some where clause is "short for" `for<'a> if (WC is well-formed) WC`. In other words, there are enough defaulted where-clauses to ensure that the where clause `WC` is well-fromed.

* `for<P> WhereClause`

### What happens if we do this?

Let's go back to our example of a caller function that didn't compile ([playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2021&gist=8f0ed5d96f022fba2e0c367edf119a45)):

```rust
fn main() {
    let x = vec![1, 2, 3];
    let s: & /*'a*/ u32 = &x[..]; // call the lifetime here 'a
    foo(s); // ERROR
}

fn foo<T>(t: T)
where
    T: for<'short> IntoData<'short>
{
    let data: &[u32] = t.into_data();
    process(data);
}
```

The problem here was that `main` had to prove that `&'a [u32]: 'short` for all `'short`, and that just wasn't true. But with these new implications, `main` has to prove something else:

```
forall<'short> {
    if (&'a [u32]: 'short) { // where clause from `IntoData`
        &'a [u32]: 'short // OK, that seems true :) 
    }
}
```

### Status of implementation

I haven't *QUITE* gotten this part modeled in mir-formality yet, there's some refactoring and tweaks needed. But actually writing out this doc helped me to see that my in-progress edits were wrong-headed anyway so that's good. 

## What about WF for higher-ranked TYPES?

OK, it's 8:54am and I'm frantically typing to finish this by 9am, but the short version is that for `for<P> T` we'll add in defaulted conditions in the same way, something like `for<P> if (T is well-formed) T`. Since mir-formality supports implication types, this should be possible.

## Questions from meeting

Feel free to add questions here, use `###` as the header.

### Question format

nikomatsakis: Wait, how do I format the questions, can you show me an example?

Yep, just like this!

### Comparing these defaults to rustc

nikomatsakis: Wait! I have a question! How do these defaults for higher-ranked trait bounds compare to what rustc does?

Answer: This is more accepting than rustc. We can tweak the definition slightly if we care to get closer. Example:

```rust
trait Foo<'a, T: Ord> () {}

fn test<A, B>()
where
    for<'z> A: Foo<'z, B>
{    
}
```

In rustc, this errors because `B: Ord` is required. Under the definition I gave, `B: Ord` would be part of the default bounds. We can tweak this by not including *all* the WF conditions as default bounds, but only those that involve the `'z` variable. In that case, `B: Ord` would not be defaulted, and the WF checking would report an error on this example, just as rustc does.

### Who can add questions

nikomatsakis: As the author of this doc, am I allowed to add questions?

Answer: Yes, damn it. WE MAKE THE RULES.

### Should we have a `(in-scope)` predicate in mir-formality?

nikomatsakis: So long as we have a more limited notion like rustc does of what we expan, I am not sure if adding default `(well-formed)` predicates is the right thing or not. I was coming up with some torturous examples where it might not behave as expected. Let me see if I can do craft one.

The idea is that you are allowed to assume that a given type is fully well-formed, even though you can't prove that yourself. (I guess the WF checking ensures that you have enough where-clauses to prove it yourself, so that's kind of its role.)

```rust

```

### in-scope and #25860

> don't we have the issue that the implied bounds are on types which are fully inferred, so any fix during rust -> hir lowering won't be enough? (though this will get fixed by implication types later on, so it doesn#t matter ^^)

You mean for closures, for example?

I think this is ok because: the thing that gets added into the environment is

```
(well-formed (type ?T))
```

where `?T` is the yet-to-be-inferred argument type. The way that elaboration works in mir-formality (bottom-up, not top-down), we won't be able to do much with that *until* we learn things about what `?T` winds up being inferred to be. At that point we would be able to.

[Zulip discussion.](https://rust-lang.zulipchat.com/#narrow/stream/326132-t-types.2Fmeetings/topic/meeting.202022-07-08/near/288941603)

### what's a biformula?

Answer: see the footnote [^biformula].

> :heart: 

### https://play.rust-lang.org/?version=stable&mode=debug&edition=2021&gist=6e27f3dbe6bb4127f615df03fe60755f shouldn't compile at all, even with `for<'short>` meaning the right thing

> we have `impl<'d> IntoData<'d> for &'d [u32]` and trait matching is invariant, isn't it? so `&'d [u32]: IntoData<'c>` doesn't hold, even if `'c: 'd` holds? we would need `impl<'c, 'd: 'c> IntoData<'c> for &'d [u32]` at which point we don't have latebound lifetimes anymore

nikomatsakis: Hmm, that depends on what the type `T`.. maybe. I was trying to come up with a good example and I wans't very happy with that one. That said, why are the lifetimes not late-bound? Which lifetimes do you even mean? 

lcnr: because we need an explicit `'d: 'c` bound

nikomatskais: There are no late-bound lifetimes on impls. I think if you change the impl, the example works as is.

let's mvoe this to zulip lol
