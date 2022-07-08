# Lazy TAIT Inference Algorithm Implications

This write-up documents the effects of the new "lazy TAIT" inference algorithm implemented in [PR #94081](https://github.com/rust-lang/rust/pull/94081). 

It begins with a write-up explaining the algorithm "from first principles" -- i.e., how a user should understand it. It then explains some of the implications, and cases where this differences from the current stabilized behavior.

## Algorithm from first principles

When you define an impl trait in "existential" position:

```rust
type Foo = impl Debug;
fn foo() -> Foo { 22_u32 }

// Or, (mostly) equivalently:
fn foo() -> impl Debug { 22_u32 }
```

this corresponds to an "opaque type" `O`. For each opaque type, the compiler has the job of determining exactly what type `T_hidden` it represents (`u32`, in our example). This is called the *hidden type* of O. `T_hidden` is called the *hidden type* because, for the most part, other code cannot rely on it. Instead, that code treats `O` opaquely -- i.e., as "some type that implements `Debug`". This means that it can determine that `O: Debug`, but not that `O: Eq`, even though `u32: Eq` is true. The only exception is with autotraits: other code can figure out that `O: Send` because `u32: Send`. This is called "auto trait leakage", and it will be discussed later.

### How the compiler infers the hidden type

The compiler infers the hidden type for an opaque type by looking at how that opaque type is used within its *defining scope*. The defining scope is defined as all code within whatever "container" declared the opaque type:

1. For type-alias-impl-trait (`type Foo = impl Debug`), the defining scope is the enclosing function, module, or impl.
2. For return-position-impl-trait (`fn foo() -> impl Debug`), the defining scope is the body of the function `foo`.

For the remainder of this discussion, we'll work with the example of a type-alias-impl-trait like so:

```rust
type Foo = impl Debug;
```

When type-checking code within the defining scope of `Foo`, we may encounter variables or values that are declared to be of type `Foo`, like the local variable `x` in this example:

```rust
type Foo = impl Debug;

fn example() {
    let x: Foo = 22_u32;
    ...
}
```

To be well-typed, this code requires that `u32` be a subtype of `Foo`. **Outside of the defining scope**, that would be an error -- but **inside** the defining scope, it is instead adopted as a constraint on what type `Foo` can represent. Therefore, `example` constrains `Foo`'s hidden type to be `u32` in this case.

The same applies to the more common case of a function whose return type is declared as an opaque type:

```rust
type Foo = impl Debug;

fn make_foo() -> Foo {
    22_u32
}
```

`make_foo` also constrains `Foo`'s hidden type to be `u32`.

Another way to get a value of an opaque type is through a recursive call, or through a call to another function within the defining scope:

```rust
type Foo = impl Debug;

fn make_foo() -> Foo {
    let mut x: Foo = make_foo();
    x = 22_u32;
    x
}

fn make_bar() -> Foo {
    let x: Foo = make_foo();
    22_u32
}
```

These functions both constrain `Foo` to be equal to `u32`.

### Incomplete constraints

When a function imposes a constraint, it must be a complete type:

```rust
type Foo = impl Debug;

fn insufficient() {
    // ERROR: Requires `Foo = Option<T>`, but what is `T`?
    let x: Foo = None;
}
```

Note though that the function can apply multiple constraints, which together suffice to fully specify the hidden type:

```rust
type Foo = impl Debug;

// Constrains `Foo = Result<u32, i32>`
fn sufficient() {
    let x: Foo = Ok(22_u32);  // Requires `Result<u32, _>`
    let y: Foo = Err(22_i32); // Requires `Result<_, i32>`
}
```

This does not work across functions. Every function must declare a full hidden type. The following example will thus not work:

```rust
type Foo = impl Debug;

fn foo() -> Foo { Ok(42) }
fn bar() -> Foo { Err(69) }
```

### Functions that impose no constraints

Even within the defining scope, it is possible for a function to reference values of the type `Foo` without imposing any constraints on them:

```rust
type Foo = impl Debug;
fn take_foo(f: Foo) {
    // Requires that `Foo: Debug`, but that is given 
    // from the declared constraints.
    println!("{f:?}");
}
```

### Multiple functions that all impose constraints

When an opaque type is defined at the module level, it is possible for there to be multiple functions which each constrain the same opaque type:

```rust
type Foo = impl Debug;

fn example() { let x: Foo = 22_u32; }      // Adds constraint: `Foo = u32`
fn make_foo() -> Foo { 22_u32 }            // Adds constraint: `Foo = u32`
fn take_foo(f: Foo) { println!("{f:?}"); } // No constraint.
```

This is allowed, so long as all of the following are true:

* all functions impose the same constraint
* each function imposes a complete constraint without type variables
* there is at least one constraint

Examples that do *not* meet those rules:

```rust
type Foo = impl Debug;

fn make_u32() -> Foo { 22_u32 }
fn make_i32() -> Foo { 22_i32 }
```

### Implementing a trait (inside *or* outside a defining scope)

When the compiler tries to decide if an opaque type `Foo` implements a trait, it does so based on the declared bounds (the one exception is auto traits; see below). The hidden type is never used. This means that we sometimes get errors where the hidden type would have worked:

```rust
type Foo = impl Debug;

fn is_display<T: Display>() { }

fn example() {
    let f: Foo = u32;
    is_display::<Foo>(); // Error: Foo is not known to be display
}
```

This can be surprising when using APIs like `Default` or `collect`:

```rust
fn foo() {
    let x: Foo = 42_i32;
    let y: Foo = Default::default(); // Error Foo: Default not satisfied
}
```

You can fix code like the above by specifying the type manually:


```rust=
fn foo() {
    let y: Foo = <i32>::default();   // OK
    
    let z: i32 = Default::default();
    let z: Foo = z;                  // Also OK
}
```
### Auto-trait leakage from *inside the defining scope*

When some code `f` inside the defining scope attempts to determine whether an opaque type like `Foo` implements an auto-trait, this is considered to be adding a "constraint" on the hidden type. In other words, to show that `Foo: Send`, `f` must constrain `Foo` to be some hidden type `H` which implements `Send`.

Example:

```rust
type Foo = impl Debug;

fn is_send<T: Send>() { }

fn not_good() {
    // Error: this function does not constrain `Foo` to any particular
    // hidden type, so it cannot rely on `Send` being true.
    is_send::<Foo>();
}

fn ok() {
    // Constrain `Foo = u32`
    let x: Foo = 22_u32;
    
    // No problem `Foo = u32` and `u32: Send`
    is_send::<Foo>();
}
```

### Auto-trait leakage

When code outside the defining scope attempts to determine whether some opaque type `Foo` implements an auto-trait, the code can "reveal" the hidden type, as shown here:

```rust
mod defining_scope {
    pub type Foo = impl Debug;
    pub fn ok() -> Foo { 22_u32 }
}

fn is_send<T: Send>() { }

fn example() {
    // OK: Reveals the hidden type for the purposes of checking `Send`
    is_send::<defining_scope::Foo>();
}
```

Revealing the hidden type forces all code in the defining scope to be type-checked; it is a fatal compilation error if this results in a cycle:

```rust
// Results in a cycle error.

fn is_send<T: Send>() { }

mod defining_scope1 {
    pub type Foo = impl Debug;
    pub fn ok() -> Foo { 
        is_send::<crate::defining_scope2::Bar>(); 
        22_u32 
    }
}

mod defining_scope2 {
    pub type Bar = impl Debug;
    pub fn ok() -> Foo { 
        is_send::<crate::defining_scope1::Foo>(); 
        22_u32 
    }
}
```

## Desirable properties of this algorithm

* Because each function must impose a complete constraint, and because auto-trait leakage within the defining scope, all functions within the defining scope are checkable independently.
    * This is good for the compiler, but it also means that users can remove a function and their code continues to compile, as long as it was not the *only* constraint.


## Differences from our current behavior

### Insta-stable: Accept recursive call sites

Consider this example:

```rust
fn bar(b: bool) -> impl std::fmt::Debug {
    if b {
        return 42
    }
    let x: u32 = bar(false); // this errors on stable
    99
}
```

On stable today, this function fails to compile:

```
error[E0308]: mismatched types
 --> src/lib.rs:5:18
  |
1 | fn bar(b: bool) -> impl std::fmt::Debug {
  |                    -------------------- the found opaque type
...
5 |     let x: u32 = bar(false); // this errors on stable
  |            ---   ^^^^^^^^^^ expected `u32`, found opaque type
  |            |
  |            expected due to this
  |
  = note:     expected type `u32`
          found opaque type `impl Debug`
```

On the new branch, that code would be accepted. The recursive call to `bar` returns the opaque type, which is then constrained to be equal to `u32`. This constraint is accepted because it occurs within the defining scope (`bar`).

## Differences between TAIT and RPIT with this algorithm

In principle, these two definitions of `foo` should be equivalent, apart from defining the type `Foo`:

```rust
mod tait {
    pub type Foo: Debug;
    pub fn foo() -> Foo { .. }
}

mod rpit {
    pub fn foo() -> impl Debug { .. }
}
```

However, crater testing revealed some cases where stable code (i.e., code using RPIT) was behaving in ways that the TAIT algorithm did not accept. Therefore, we added some "special cases" for RPIT notation. We need to decide (in follow-up PRs) how to align the behavior of TAIT and RPIT in these cases. **The current choices were made to allow for maximum future flexibility.**

### Return statement

`return` statements and the trailing return expression are special with RPIT (but not TAIT). So while this TAIT program fails to compile

```rust
#![feature(type_alias_impl_trait)]
type Foo = impl std::fmt::Debug;

fn foo(b: bool) -> Foo {
    if b {
        return vec![42];
    }
    std::iter::empty().collect() //~ ERROR `Foo` cannot be built from an iterator
}

```

The equivalent RPIT code works just fine:

```rust
fn bar(b: bool) -> impl std::fmt::Debug {
    if b {
        return vec![42]
    }
    std::iter::empty().collect() // Works, magic
}
```

The RFCs do not mention such cases and there are no tests in the Rust test suite excercising this behavior.

Note that in the new insta stable case mentioned earlier, when we are working with the return value of a recursive call, both RPIT and TAIT get errors (on this branch and later):

```rust
type Foo = impl std::fmt::Debug;

fn foo(b: bool) -> Foo {
    if b {
        return vec![];
    }
    let mut x = foo(false);
    x = std::iter::empty().collect(); //~ ERROR `Foo` cannot be built from an iterator
    vec![]
}

fn bar(b: bool) -> impl std::fmt::Debug {
    if b {
        return vec![];
    }
    let mut x = bar(false);
    x = std::iter::empty().collect(); //~ ERROR `impl Debug` cannot be built from an iterator
    vec![]
}
```

### Branches

Similar from the user perspective but very different on the compiler side, TAITs do not allow type inference across branches of an `if` or `match`:

```rust
type Foo = impl std::fmt::Debug;

fn foo(b: bool) -> Foo {
    if b {
        vec![42_i32]
    } else {
        std::iter::empty().collect()
        //~^ ERROR `Foo` cannot be built from an iterator over elements of type `_`
    }
}
```

The equivalent RPIT example works just fine.

It is easy to support, but we should make an explicit decision to include the additional complexity in the implementation (it's not much, see a721052457cf513487fb4266e3ade65c29b272d2 which needs to be reverted to enable this).

For closures the opposite is true:

```rust
fn bar(b: bool) -> impl std::ops::FnOnce(String) -> usize {
    if b {
        |x| x.len() //~ ERROR type annotations needed
    } else {
        panic!()
    }
}
```

does not work on stable, even though

```rust
fn bar1(b: bool) -> impl std::ops::FnOnce(String) -> usize {
    |x| x.len()
}
```

does work.

The equivalent code with TAIT works just fine with lazy TAIT, because the opaque type stays opaque right until it is compared against the closure, thus giving the maximum amount of information to the closure.

```rust
type Foo = impl std::ops::FnOnce(String) -> usize;

fn foo(b: bool) -> Foo {
    if b {
        |x| x.len()
    } else {
        panic!()
    }
}


type Foo1 = impl std::ops::FnOnce(String) -> usize;
fn foo1(b: bool) -> Foo1 {
    |x| x.len()
}
```