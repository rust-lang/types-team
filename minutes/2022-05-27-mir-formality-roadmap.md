# Formality roadmap

## Goals

* Complete model of safe Rust including all static checks + operational semantics
* Able to run against portions of standard Rust test suite to compare behavior

## Current status of MIR formality

* Subtyping algorithm modeled
* Solver is "explore all routes", does not match recursive structure in chalk
* Many gaps along the way:
    * ... (fill in)

## Big picture tasks and layers

* *Checking a MIR program:* The **mir** layer drives the process of checking that a program meets all of Rust rules. This relies on...
    * *OK goals*: the **decl layer** creates an "ok goal" for the various Rust declarations
    * *Type checker*: The **mir** layer 
* *Solving goals*: The **logic layer** is tasked with taking a goal and returning a answer scheme. This relies on...
    * *Defining what is true:* The **decl layer** is tasked with converting Rust declarations into *clauses*
    * *Built-in Rust rules:* The **ty layer** implements various built-in relations like subtyping

## Spikes

The plan is to drive this development through *spikes*, meaning example tests that we can use to work out what parts of the sytem need to be implemented. nikomatsakis intends to have enough implemented by next Wednesday's formality meeting that folks can start filling in.

Spikes and why they were chosen:

* Issue #25860: https://github.com/nikomatsakis/a-mir-formality/issues/22
    * validates our fix to a long-standing soundness problem
* Lending iterator: https://github.com/nikomatsakis/a-mir-formality/issues/40
    * important to pending stabilization
* TAIT: 
    * pending stabilization, but also raises some interesting questions

## Test driver

Filed under https://github.com/nikomatsakis/a-mir-formality/issues/42

> We should create a test harness / rustc driver `formalityc` that works as follows:
>
> * It parses an input rust crate and compiles it to MIR
> * It generates MIR formality files from the MIR and some portion of the interfaces of other crates
> * It writes those files to a test file and then invokes `racket` to run MIR formality against them
>     * It reads some sort of special comments (e.g., `// compile-fail`) to determine what behavior is expected.
>
> This can be used to extend our test suite with Rust test files rather than hand-coding things in Racket.
>
> Eventually I expect the racket code to go away and be replaced with either Rust code or (my current preference) some Rust interpreter of our definition.


## Implementation in rustc

Another angle is ongoing work to align the formality model with rustc implementations:

* subtyping work

## Discussion: Implementation in chalk

Another angle is ongoing work to align the formality model with chalk implementations:

* Formality does not model the chalk recursive trait solver
* Chalk needs to adopt Formality's overall goal structure
* Chalk needs to adopt Formality's subtyping and type-relating goals
* Chalk needs to adopt Formality's approach to associated types

Conclusion from discussion: probably we just want to restart chalk from scratch.

## Discussion: Overview of rustc static checks

Key:

* :x: never expect to model this
* :hourglass: would like to model this, but not right now
* :heavy_minus_sign: not covered by the spikes above
* :heavy_check_mark: this is part of our first goal

| Phase | Modeled? | How? |
| --- | --- | --- |
| Parsing | :x:
| Macro expansion | :hourglass: 
| Name resolution | :hourglass: 
| Lifetime elision rules | :hourglass: 
| Associated type trait selection | :hourglass: | e.g. `T::Item` becoming `<T as Iterator>::Item`
| Variance inference | :hourglass:  | we will take variance as "input" to the declarations |
| Coherence overlap / orphan | :heavy_check_mark:  | The decl layer's OK check |
| HIR type checker's WF checks | :heavy_check_mark:  | The decl layer's OK check |
| Dyn safety rules | :heavy_minus_sign: | 
| Impl matches trait? | :heavy_check_mark: | Decl layer's OK check|
| Const promotion | :hourglass: | decided by MIR building in rustc 
| HIR type checking of fn bodies | :hourglass:  | we take MIR as input |
| Method resolution | :hourglass: | decided by HIR type checker, embedded in MIR |
| Closure capture inference | :hourglass: | decided by HIR type checker, embedded in MIR |
| Closure mode (fn, fnmut) and signature inference | :hourglass: | decided by HIR type checker, embedded in MIR |
| Coercion insertion | :hourglass: | decided by HIR type checker, embedded in MIR |
| Drop/StorageDead placement | :hourglass: | decided by MIR building in rustc |
| Patterns/exhaustiveness | :hourglass: | decided by MIR building in rustc |
| Unsafe checking | :heavy_minus_sign: | Part of "type checking" on MIR |
| MIR type check | :heavy_check_mark: | mir layer generates goals 
| Borrow checking | :heavy_check_mark: | mir layer polonius-style check |
| Rust subtyping | :heavy_check_mark: | modeled by the ty layer and also decl |
| Operational semantics | :hourglass: | basically what miri does |
| Transmute rules | :hourglass: 
| Layout | :hourglass: | input to operational semantics for now |

## Discussion: Const generics

Discussion around whether we want to model const generics and what would be required to do so.

Conclusion was that we needed deeper discussion, but some felt modeling const generics could be useful for resolving some of the questions around newer features; this would broaden scope to include 

## Questions for discussion

* What about coherence rules?
* What is needed to align chalk and formality?
* Should we include operational semantics and -- if so -- how does that fit in?
    * conclusion from meeting *seemed* to be to keep it out of scope to start
* ~~What are all the static checks rustc does, and which are covered by the spikes above?~~
    * see table
* Const generics?