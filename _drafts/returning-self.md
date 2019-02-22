---
layout: post
title: "Returning Self Considered An Anti-Pattern"
categories: rust
---

# Returning `Self` Considered ~~Harmful~~ An Anti-Pattern

It's a common pattern in the Rust ecosystem to have a function return some variation of `Self` (either `Self`, `&Self`, or `&mut Self`) in order to enable method chaining. For example:

```rust
#[derive(Default)]
struct Foo;

impl Foo {
    fn chain(self) -> Self {
        self
    }
}

let foo = Foo::default()
    .chain()
    .chain()
    .chain()
    .chain();
```

This pattern is often found in combination with the [builder pattern], where you have a struct containing a number of optionally-configurable values, enabling the user to concisely set only the options they care about.

While method chaining is often desirable as a concise way to perform a series of operations, returning `Self` interacts poorly with Rust's ownership system.

## Use Cases

If you're writing a crate that provides a builder struct, you'll want to design your API so that it can support all of the following use cases:

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
foo.chain()
```

---

To demonstrate this, we're going to work with the following definitions:

```rust
#[derive(Default)]
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

### Single Method Chain

The most basic case is having a single long method chain, from construction into the consumtion of your type.

```rust
consume_move(Foo::default().chain_move().chain_move());
```

```rust
consume_ref(Foo::default().chain_ref().chain_ref());
```

For both the ref and move versions, a single large chain works as expected, and you can pass the output directly into the consuming function.

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

```
TODO: Error goes here.
```

You instead need to assign the initially constructed `Foo` to a variable, then perform the method chaining:

```rust
let mut foo = Foo::default();
foo.chain_ref().chain_ref();
println!("foo: {:?}", foo);
consume_ref(&foo);
```

While this works, it's less ergonomic and somewhat undermines one of the major advantages of method chaining (i.e. being able to create, modify, and assign the value in a single statement).

---

Template gist: https://gist.github.com/6aa2a0992ed5043e72ed804e5f221101

{% rp_highlight rust %}
fn main() {
  println!("{}", "Rust Is Memory Safe!"};
}
{% endrp_highlight %}
