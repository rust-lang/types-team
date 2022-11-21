# Chalk integration proposal (2022-09-30)

Zulip discussion link: https://rust-lang.zulipchat.com/#narrow/stream/326132-t-types.2Fmeetings/topic/2022-09-30.20chalk.20integration/near/301641085

## Meta comment and plan

We've had trouble "getting going" on chalk integration and there are various ongoing efforts. Niko's meta proposal for how to proceed is as follows:

* We create a living design repository that will describe the design we are working towards
    * either as part of a-mir-formality, as a fork of rustc-dev-guide, or as its own thing
    * we review updates to this design repository on a regular basis
    * this hackmd is meant as the start of that material
* We prototype the final design in a-mir-formality
    * e.g. this is a good forum to explore some of the questions around lifetime inference
    * this hackmd links 
* We refactor rustc to bring it closer to the overall design, oncovering constraints as we go
    * this is also ongoing, and is the third part of this document (very sketchy)

## "The chalk vision"

Chalk is a kind of shorthand for a style of solver that can be both efficient, correct, and shared across multiple codebases. This will likely build on some parts of today's chalk repo, but also integrate lessons from a-mir-formality and other work that has happened in the meantime.

This section aims to explain the proposed high-level architecture. The a-mir-formality repository models this architecture quite closely, and so we have links in the doc to that repository as well.

### Various "actors"



### Core interface to the database

Logic solver and type relations code can execute various callbacks:

* `fn generics_of(&self, def_id: DefId) -> Vec<GenericParameter>` 
* `fn predicates_of(&self, def_id: DefId) -> Vec<Predicate>`
* `fn clauses_for_goal(&self, goal: &Goal) -> Vec<Clause>`
    * convert from Rust syntax into logical clauses
    * e.g., `impl<T> Foo for Vec<T> where T: Debug` might get translated to `has_impl(Vec<T>: Foo) :- is_implemented(T: Debug)`

In a-mir-formality, these are the ["hook" methods implemented by the decl layer](https://github.com/nikomatsakis/a-mir-formality/blob/038b0e1ccfb2a3f591688d5855a42c129d577e85/racket-src/decl/env.rkt#L65-L85).

### Core interface to/from the type relation code

```rust
fn relate(
    db: &Db, 
    env: &mut Env, 
    relation: Relation,
) -> Result<Vec<Goal>, Error>;
// Yields Ok(goals) with subgoals to prove,
// or Err(_) if this is not provable at all.

enum Relation {
    t1: ty::Generic,
    op: RelationOp,
    t2: ty::Generic,
}

enum RelationOp {
    SubtypeOf,
    Outlives,
}
```

In a-mir-formality, this is the "ty" layer, and `relate` is [`ty:relate-parameters`](https://github.com/nikomatsakis/a-mir-formality/blob/038b0e1ccfb2a3f591688d5855a42c129d577e85/racket-src/decl/env.rkt#L74).

### Core interface to/from the solver

```rust
fn perform_query(db: &Db, canonical_query: CanonicalGoal) -> Response

enum Response {
    // Known to be true. The substitution
    // constraints the input variables.
    True(Substitution, VarsAndRelations),
    Maybe(Substitution),
    False,
}

struct VarsAndRelations {
    /// new inference variables 
    inference_vars: Vec<(Universe, Name)>,
    
    // new relations: these are low-level
    // relations, primarily `'a: 'b`
    relations: Vec<Relation>,
}
```

Where a `CanonicalGoal` is

* Bound variables (inference, universal placeholders)
* Parameter environment
* Goal expressed as a logical predicate:
    * `is_implemented`
    * `normalizes_to` etc

and a `Response` includes

* the core result (yes/no/maybe)
* a substitution of things that are known to be true
    * even with a maybe result, we may learn something about the inference variables
* relations that have to be checked later by the region code
    * you can think of this as a list of outlives obligations on region variables
    * e.g. `T: Foo<'a>` might be true "if `'a: 'static`"
    * in a-mir-formality, subtyping and other relations can be returned as well, but we have already checked for overall consistency; this has more to do with the simple-sub inference algorithm, which is really a separate consideration worthy of more discussion.

This is the "logic" layer of a-mir-formality, and especially [the `logic:solve-query` function](https://github.com/nikomatsakis/a-mir-formality/blob/main/racket-src/logic/solve-query.rkt#L15-L17).

#### The recursive solver

Chalk includes multiple solvers that can fulfill the above interface. The choice of solver does affect what kinds of programs compile and so it is part of the specification, though we have room to improve it over time. For various reasons, nikomatsakis thinks that the recursive solver is the best choice at the moment. It effectively works like this:

```rust
fn solve(canonical_goal: CanonicalGoal) -> Result<()> {
    // get the goal to prove, e.g., `is_implemented(Vec<?T>: Debug)` or
    // `projected_to(<?T as )
    let (inference_context, goal) = InferenceContext::new(canonical_goal);
    
    // 
    let mut subgoals: Vec<_> = find_subgoals(goal);
    while !subgoals.is_empty() {
        let subgoal = subgoals.pop();
        let canonical_subgoal = inference_context.canonicalize(subgoal);
        match solve(canonical_subgoal) {
            True(solution) => inference_context.include(solution),
            False => return Err,
            Maybe(solution) => {
                inference_context.include(solution);
                subgoals.push(subgoal);
            }
        }
    }
    True(inference_context.make_solution())
}
```


### Managing diagnostics

This part is less well-developed. The basic idea is to offer a second interface to the solver:

```rust
fn perform_query_tree(db: &Db, canonical_query: CanonicalGoal) -> ResponseTree
```

that solves the query but constructs and returns the resulting tree, showing what things it tried to prove and why they failed. This could be implemented separately or with one codebase that is generic over which mode it is.

### How this is expected to be integrated into rustc

#### Unifying fulfill, evaluation, selection, and projection into one mechanism

Currently we have four bits of code: 

* fulfillment context -- tracks what has to be proven
* selection -- selects what impl to use
* evaluation -- determines if a query might be true
* projection -- normalizes associated variables

In the proposed approach, all of these are unified into performing queries (with possible exception of the first). 

#### Caching, cycles, and incremental compilation

The chalk "recursive solver" can be integrated quite cleanly with the rust query system, if we extend it to support "fixed point" queries. The idea is that each of the methods identified above can be considered a query. Using the recursive solver style from chalk, the solver creates new subqueries for each subgoal. Each subquery returns a solution that summarizes all possible solutions to the subgoal.

Effectively the recursive solver works like this:

```rust
fn solve(canonical_goal: CanonicalGoal) -> Result<()> {
    let (inference_context, goal) = InferenceContext::new(canonical_goal);
    let mut subgoals: Vec<_> = find_subgoals(goal);
    while !subgoals.is_empty() {
        let subgoal = subgoals.pop();
        let canonical_subgoal = inference_context.canonicalize(subgoal);
        match solve(canonical_subgoal) {
            True(solution) => inference_context.include(solution),
            False => return Err,
            Maybe(solution) => {
                inference_context.include(solution);
                subgoals.push(subgoal);
            }
        }
    }
    True(inference_context.make_solution())
}
```


#### HIR type checking

HIR type checker would make use of a pared down, simplified fulfillment context that just tracks the top-level obligations that are not yet solved:

```rust
struct FulfillmentCx {
    goals: Vec<Obligation>
}
```

the equivalent of `select_where_possible` etc would be to:

* iterate the list and perform queries
    * queries returns suggested unifications, incorporate those
    * if this yields back true, remove it from the list

Note that the same top-level goal may correspond to different queries each time, as it may reference inference variables that have since been constrained.

When an error occurs -- either an obligation that cannot be proven or goals that remain ambiguous once all type checking is complete -- we would rerun those obligations in diagnostic mode, yielding a detailed trace, and then figure out how to report that to user.

The HIR type checker, when run with `-Zchalk`, already does something kind of like this (but without the smart handling for diagnostics).

#### Lazy normalization

Lazy normalization means that associated types are not normalized until they are required to be. This would help avoid quite a number of cycles in 

#### MIR type checking, borrow checker

The MIR type checker already uses a query system, but it doesn't require the full generality of the above because (at least for now...) it doesn't expect type inference variables.

One key difference in the chalk model that is not obvious from the interfaces sketched above is the way that it handles higher-ranked outlives obligations like `for<'a> (T: 'a)`. In rustc today, the obligations that result from a trait operation can include higher-ranked obligations like this, and they are ultimately resolved by the borrow checker.

In MIR formality, higher-ranked obliations are handled by the type checker, and the borrow checker only gets simple constraints between variables in the "root universe". This is a much better fit for a system like polonius, which is ill-equipped to deal with higher-ranked reasoning.

#### Codegen

Codegen uses the trait solver in a few particular ways. The primary one is to figure out which version of a trait method is being called. In some cases, 

## How to get there?

Here are some ideas for initial projects.

### chalk-ir crate

Part of the plan is to have chalk be extracted from rustc 

### "Shallow subtyping"

The current subtyping code produces higher-ranked obligations like `for<'a> '0: 'a`. The goal here is to process those at the trait solving layer (after all, all the bounds of `'a` are known up-front), thus freeing the borrow checker from the need to think about regions. This is a blocker for polonius integration as well.

### Diagnostics query (proposed by lcnr)

Begin by modifying the type checker to integrate

### Richer existential types in rustc



### Implication predicates

### Coherence

Implement the new solver and integrate it into coherence. The modeling of coherence in a-mir-formality is complete apart from specialization and negative impls.

Coherence is interesting because it is only using an "evaluation" style testing.

## Some question marks for further exploration

There are some things that need more exploration in a-mir-formality.

### Lifetime erasure

We have to think about which lifetimes we erase from queries precisely and how we can guarantee lifetime erasure going forward. 

### Existential predicates

To properly model TAITs, we need

# Questions

## Coordinating overal effort and tracking the overall design

nikomatsakis: My sense is that there is a need to get consensus on the overall design we are working for. I am proposing the drafting of a design doc and continued prototyping work on a-mir-formality as a way to drive that design. I feel like trying to do all our discussion and experimentation by live editing rustc code is going to slow us down, and that we need to be simultaneously doing that *and* working "up ahead". I also feel like my own personal time is probably best spent "up ahead", which is why i've been focused on hacking on a-mir-formality. But I'd like to hear what others think on this point and how to get that communication going in both directions. Should we have a dedicated "chalkification" meeting time? Smaller efforts that coordinate?

## Missing pieces that show up in rustc etc

nikomatsakis: I'm wondering what are some of the missing pieces that cause big interactions in rustc but are lacking in the models and design docs. Const generics comes to mind, are there others?

lcnr: there's codegen/ctfe which uses erased lifetimes and should still result in the same evaluation result as mentioned above. idk, do we have a doc which states all of the requirements we have on our trait system somewhere?

## Design doc points that came up

* Where does the code that converts from HIR definitions into logical predicates live ("decl" layer of a-mir-formality).
    * What is the interface that it uses to talk to rustc/rust-analyzer?

## What do we want to do with Chalk?

* There is some amount of code that might be reusable in a new trait solver implementation (in Rust) - type folding, interner interface, maybe most of chalk-ir.
* Rust-analyzer currently uses it heavily - we should consider the medium-term, where we heavily work on a-mir-formality, but rust-analyzer wants something more "stable". Are we open to making Chalk more "experimental" for things like const-eval or lifetime constraint generation under the *current* type model?

## When is a-mir-formality "ready" to move to rust-lang

We should announce that along with types team and types team efforts in a blog post :)
