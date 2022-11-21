# 2022-09-23 deep dive

Zulip discussion link: https://rust-lang.zulipchat.com/#narrow/stream/326132-t-types.2Fmeetings/topic/2022-09-23.20RPITIT.20Support.2C.20Refactoring/near/300356358

## RPIT refactor

given this...

```rust
fn bar<'a>(&'a self) -> impl Debug + 'a
```

we desugar this to

```rust
fn foo<'a>(&'a self) -> Foo<'static, 'a> {
    type Foo<..., 'a1>: Debug + 'a1;
```

because it inherits the parameters, we have to add `'static` values

Q: I guess one of the questions could have been ... why this is not just `Foo<'a1>` and you forget about what is not being captured?

A: the short version: that's what we were trying to do :)

Example:

```rust
fn foo<'a, T>(a: T) -> impl Debug
where
    T: SomeTrait<'a> + Debug
{
    a
}
```

`'a` not captured, but appears in where clauses; `T` captured