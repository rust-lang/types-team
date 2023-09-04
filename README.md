# Types Team

## Scope and purpose

The **types** team is dedicated to improving the trait
system implementation in rustc. This team is a collaboration
between the [lang team] and the compiler team. We have a number of inter-related
goals:

- designing new trait-related language features;
- documenting and specifying the semantics of traits in Rust today; and,
- improving the trait solver implementation in rustc.

[lang team]: https://github.com/rust-lang/lang-team/

A big part of this work is transitioning the compiler to use a
[Chalk-style] solver, but along the way we hope to make targeted fixes
to the existing solver where needed.

[Chalk-style]: https://github.com/rust-lang-nursery/chalk

## Design meetings

We hold weekly design meetings where we talk in depth about various
topics ([calendar event][ce-design]).  These meetings take place on Zulip (see below). The goal is
not just to figure out what we want to do, it's also a way to spread
knowledge. Feel free to come and lurk!

[ce-design]: https://calendar.google.com/calendar/u/0/embed?src=6u5rrtce6lrtv07pfi3damgjus@group.calendar.google.com

You'll find minutes from past meetings in [the minutes directory](minutes).

## Chat forum

On [the rust-lang Zulip][z], in [the `#t-types` stream][s].

[z]: https://rust-lang.zulipchat.com/
[s]: https://rust-lang.zulipchat.com/#narrow/stream/144729-t-types

## Dedicated repository

Documents related to the types team are stored on a
dedicated repository, [rust-lang/types-team]. This repository contains
meeting minutes, past sprints, as well as draft RFCs and other
documents.

[rust-lang/types-team]: https://github.com/rust-lang/types-team
