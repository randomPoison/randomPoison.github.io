---
layout: post
title: "Returning Self Considered An Anti-Pattern"
categories: rust
---

It's a common pattern in the Rust ecosystem to have a function return some variation of `Self` (either `Self`, `&Self`, or `&mut Self`) in order to enable method chaining. For example:

```rust
#[derive(Default)]
struct Foo;

impl Foo {
    fn chain(self) -> Self {
        self
    }
}

fn consume(foo: Foo) {}

// Create, modify, and consume a `Foo` in a single expression.
// So concise! Such ergonomic! Wow!
consume(
    Foo::default()
        .chain()
        .chain()
        .chain()
        .chain()
);
```

> [Run in the Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2018&gist=474c0928fe2620c4a889378f46558336)

This pattern is often found in combination with the [builder pattern], where you have a struct containing a number of optionally-configurable values, enabling the user to concisely set only the options they care about.

While method chaining is often desirable as a concise way to perform a series of operations, returning `Self` interacts poorly with Rust's ownership system.

# Why Returning `Self` Doesn't Work Well

While the above example demonstrates the most straightforward way of doing method chaining (i.e. initializing and modyfing an object in a single statement), there are oftem more complex use cases that don't work nearly as well.

If your struct `Foo` defines a method `chain()` intended to be used in method chaining, here are some different ways a user may want to use it:

Performing the method chain in a single expression:

```rust
let foo = Foo::default()
    .chain()
    .chain()
    .chain()
    .chain();
```

Splitting up the method chain with re-assignment:

```rust
let foo = Foo::default()
    .chain()
    .chain();

let foo = foo
    .chain()
    .chain();
```

Splitting up the method chain without re-assigment:

```rust
let mut foo = Foo::default()
    .chain()
    .chain();

foo
    .chain()
    .chain();
```

Calling chain functions individually:

```rust
let mut foo = Foo::default();
foo.chain();
foo.chain();
foo.chain();
```

Neither returning `Self` or `&mut Self` supports all of these cases: Returning `Self` means that you can't call the `chain` function on its own without re-assigning the variable (i.e. you have to do `let foo = foo.chain()`); Returning `&mut Self` means that you can't split up the chain without also splitting off the intial creation of the object, e.g.

```rust
// First you have to create the initial `Foo`.
let mut foo = Foo::default();

// Then you can do the first bit of method chaining.
foo.chain()
    .chain();

// Now you can do more chaining.
foo
    .chain()
    .chain();
```

---

To demonstrate this, we're going to work with the following definitions:

```rust
#[derive(Debug, Default)]
struct Foo;

impl Foo {
    fn chain_move(self) -> Self {
        self
    }

    fn chain_ref(&mut self) -> &mut Self {
        self
    }
}

fn consume_move(_foo: Foo) {}

fn consume_ref(_foo: &Foo) {}
```

This allows us to compare the two primary ways that people implement method chaining: Returning `Self`, and returning `&mut Self`.

Let's now take a look at each of the use cases we would like to support, and see how they work with each of the method chaining approaches.

## Single Method Chain

The most basic case is having a single long method chain, from construction into the consumtion of your type.

```rust
consume_move(Foo::default().chain_move().chain_move());
```

```rust
consume_ref(Foo::default().chain_ref().chain_ref());
```

For both the ref and move versions, a single large chain works as expected, and you can pass the output directly into the consuming function.

## Use Before Consuming

The ref version doesn't work quite as well if you need to bind the chained value to a varible before consuming it, though. Let's say you needed to log the value of `Foo` before consuming it:

```rust
let foo = Foo::Default().chain_move().chain_move();
println!("foo: {:?}", foo);
consume_move(foo);
```

```rust
let foo = Foo::default().chain_ref().chain_ref();
println!("foo: {:?}", foo);
consume_ref(foo);
```

This won't compile:

```txt
TODO: Error goes here.
```

You instead need to assign the initially constructed `Foo` to a variable, then perform the method chaining:

```rust
let mut foo = Foo::default();
foo.chain_ref().chain_ref();
println!("foo: {:?}", foo);
consume_ref(&foo);
```

While this works, it's less ergonomic and somewhat undermines one of the major advantages of method chaining (i.e. being able to create, modify, and assign the value in a single statement). For my purposes, I consider this "not working" with regards to method chaining.

## Modifying a Bound Value

Is it possible to modify an existing, mutable value via method chaining? It's possible with the `chain_ref` variant:

```rust
let mut foo = Foo::default();
foo.chain_ref().chain_ref().chain_ref();
consume_ref(&foo);
consume_move(foo);
```

The `chain_move` version requires that you rebind the variable, though, which I consider to be an ergonomic failure:

```rust
let foo = Foo::default();
let foo = foo.chain_move().chain_move().chain_move();
```

Note that if you have a `&mut Foo`, the `chain_move` version doesn't work at all, and you'll be left with no way to modify the value.

On a similar note, the `move_ref` version can also be used to modify the value without chaining, e.g.:

```rust
let mut foo = Foo::default();
foo.chain_ref();
foo.chain_ref();
foo.chain_ref();
```

The `move_self` version again requires the variable to be re-bound in each statement, and cannot be used if you only have access to a `&mut Foo`.

---

Template gist: https://gist.github.com/6aa2a0992ed5043e72ed804e5f221101
Complete demo: https://play.rust-lang.org/?version=stable&mode=debug&edition=2018&gist=1bf8a2fe6044498841aeabb223fab6f9

Results for `chain_move`:

|                    | `consume_move` | `consume_ref` |
|--------------------|----------------|---------------|
| Single chain       | yes            | yes           |
| Use before consume | yes            | yes           |
| Modify bound value | no             | no            |

Results for `chain_ref`:

|                    | `consume_move` | `consume_ref` |
|--------------------|----------------|---------------|
| Single chain       | no             | yes           |
| Use before consume | no             | no            |
| Modify bound value | yes            | yes           |

[builder pattern]: https://github.com/rust-unofficial/patterns/blob/master/patterns/builder.md
