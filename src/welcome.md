# Welcome to the traits working group

## Scope and purpose

The **traits** working group is dedicated to improving the trait
system implementation in rustc. This working group is a collaboration
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

[ce-design]: https://calendar.google.com/event?action=TEMPLATE&tmeid=MnFmbmdkaGV0aXE3Zjc4cjlpNWVjNDRoZXMgNnU1cnJ0Y2U2bHJ0djA3cGZpM2RhbWdqdXNAZw&tmsrc=6u5rrtce6lrtv07pfi3damgjus%40group.calendar.google.com

You'll find minutes from past meetings in [the minutes
directory](minutes). We also keep have a
[hackmd](https://hackmd.io/nrhN5A8sR2eY9UdfJTKcJg?edit) where we keep
a log of what's been happening and track possible agendas for upcoming
meetings.

## Chat forum

On [the rust-lang Zulip][z], in [the `#wg-traits` stream][s].

[z]: https://rust-lang.zulipchat.com/
[s]: https://rust-lang.zulipchat.com/#narrow/stream/144729-wg-traits
