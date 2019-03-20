---
layout: post
title: "Chaining Functions Without Returning Self"
categories: rust
---

<!-- markdownlint-disable MD002 -->

It's a common pattern in the Rust ecosystem to have a function return some variation of `self` at the end of a function in order to enable method chaining. For example:

```rust
// Create, modify, and consume a `Foo` in a single expression.
// So concise! Much ergonomic! Wow!
consume(
    Foo::default()
        .chain()
        .chain()
        .chain()
        .chain()
);

// Method definitions that make this possible:
// -------------------------------------------

#[derive(Default)]
struct Foo;

impl Foo {
    fn chain(self) -> Self {
        // Make some changes to `self`, then return `self`.
        self
    }
}

fn consume(foo: Foo) {}
```

> [Run in the Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2018&gist=474c0928fe2620c4a889378f46558336)

This pattern is often found in combination with the [builder pattern], thereby enabling the user to concisely set only the options they care about.

While the above example demonstrates the most straightforward way of doing method chaining (i.e. initializing and modifying an object in a single statement), there are often more complex use cases that don't work nearly as well.

In this post, I intend to demonstrate the following points:

* Returning `self` is not an effective way of achieving method chaining in Rust.
* Method and function chaining should be orthogonal to the return type of a function.
* You should only return `self` from a function if it's semantically meaningful to do so.
* Cascading and pipelining offer promising alternatives to returning `self` when you want method chaining.

## Chaining by Returning `self`

Since returning `self` is currently the de facto way of enabling method chaining in the Rust ecosystem, I'm going to start by demonstrating that doing so doesn't work as well as we would like. To show this, we're going to work with the following definitions:

```rust
// Define a struct `Foo` with some internal state that the methods
// will modify. We'll use `Foo::default()` throughout the examples
// to create the initial instance of the data.
#[derive(Debug, Default)]
struct Foo {
    value: usize,
}

impl Foo {
    // Define a method that can be chained by taking and returning
    // ownership of the data.
    fn chain_move(mut self) -> Self {
        self.value += 1;
        self
    }

    // Define a method that can be chained on a borrow of the data.
    fn chain_ref(&mut self) -> &mut Self {
        self.value += 1;
        self
    }
}

// Define a function that will consume the final data by taking
// ownership of it.
fn consume_move(foo: Foo) {
    println!("{:?}", foo);
}

// Define a function that will consume the final data by borrowing it.
fn consume_ref(foo: &Foo) {
    println!("{:?}", foo);
}
```

In the examples I will also sometimes use an imaginary method `chain` to demonstrate an idealized way of performing method chaining. This will be used to show the "ideal" use case (i.e. the most ergonomic way of applying method chaining in a given situation) so as to compare how `chain_ref` and `chain_move` work in practice.

Let's now take a look at each of the use cases we would like to support, and see how they work with each of the method chaining approaches.

### Single Method Chain

The most basic case is having a single long method chain, from construction into the consumption of your type:

```rust
consume(Foo::default().chain().chain());
```

This works reasonably well with both `chain_move` and `chain_ref` so long as you match the chaining style with the consumer:

```rust
consume_move(Foo::default().chain_move().chain_move());
consume_ref(Foo::default().chain_ref().chain_ref());
```

> [Run in the Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2018&gist=9402739e03bf725420ad70a0283ac943)

Note, though, that while the `chain_move` version works with both `consume_move` and `consume_ref` version, `chain_ref` can only be used with `consume_ref`. If we try to pass the result of `chain_ref` into `consume_move`, we get this error:

```txt
error[E0308]: mismatched types
 --> src/main.rs:9:18
  |
9 |     consume_move(Foo::default().chain_ref().chain_ref()); // Doesn't compile.
  |                  ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^ expected struct `Foo`, found &mut Foo
  |
  = note: expected type `Foo`
             found type `&mut Foo`
```

> [Run in the Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2018&gist=876fb51a394c47b29a4baa1aefd6f822)

Since `chain_ref()` returns a `&mut Foo`, we can't use it anywhere a `Foo` is expected (though there are ways of working around this, which will be covered below).

### Use Before Consuming

Let's say you needed to log the value before consuming it. The most intuitive way of doing this would be to directly bind the result of the chain to a variable, log the variable, then pass the variable to the consume method:

```rust
let foo = Foo::default().chain().chain().chain();
println!("foo: {:?}", foo);
consume_ref(&foo);
consome_move(foo);
```

This can be done directly with `chain_move`:

```rust
let foo = Foo::default().chain_move().chain_move();
println!("foo: {:?}", foo);
consume_ref(&foo);
consume_move(foo);
```

> [Run in the Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2018&gist=c3dad670cc24c7d710fc2cc4278c691c)

But doing the same thing with `chain_ref` won't compile:

```rust
let foo = Foo::default().chain_ref().chain_ref();
println!("foo: {:?}", foo);
consume_ref(foo);
```

```txt
error[E0716]: temporary value dropped while borrowed
 --> src/main.rs:2:15
  |
2 |     let foo = Foo::default().chain_ref().chain_ref();
  |               ^^^^^^^^^^^^^^                        - temporary value is freed at the end of this statement
  |               |
  |               creates a temporary which is freed while still in use
3 |     println!("foo: {:?}", foo);
  |                           --- borrow later used here
  |
  = note: consider using a `let` binding to create a longer lived value
```

> [Run in the Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2018&gist=fcf0b2d4b3422d5151fde27df66637ae)

As the compiler helpfully notes, the temporary value created by `Foo::default()` is dropped at the end of the chain sequence, so we can't bind it to a variable. Instead, we must first create the initial `Foo` and bind it to a mutable variable. Once that's done, we are able to use `chain_ref` to apply modifications to it before logging and consuming the final value:

```rust
let mut foo = Foo::default();
foo.chain_ref().chain_ref();
println!("foo: {:?}", foo);
consume_ref(&foo);
consume_move(foo);
```

> [Run in the Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2018&gist=234824151a664395c788cac897c49a9e)

Note that you must also use this approach to use `chain_ref` in combination with `consume_move`; By binding the initial `Foo` to a variable, you avoid the issue of it being a temporary value and being dropped too early.

While this is functional, it has a few drawbacks as compared to the `chain_move` version:

* You can no longer create and modify the value in a single expression.
* You have to bind `foo` as a mutable variable, which loosens some of the guarantees you get in the `chain_move` version when binding the variable immutably.
* When converting from the single chain version to this version, it's easy to initially apply the naïve transformation shown above and get tripped up when it doesn't work. The `chain_move` version, on the other hand, works fine with the naïve transformation.

For this case, both `chain_ref` and `chain_move` work equally well with `consume_ref` and `consume_move` since, once the object is bound to a variable, it is easy to either lend that value to another function or to transfer ownership entirely.

### Modifying an Owned Value

Now let's say that you want want perform an initial method chain, then conditionally apply another chain of operations to the same object. This means that we already have a bound, mutable variable that we would like to modify in the same method-chaining style that we use to create the object. The ideal version of this would be as follows:

```rust
let mut foo = Foo::default().chain().chain().chain();
if some_condition {
    foo.chain().chain().chain();
}
consume_ref(&foo);
consume_move(foo);
```

In this case, the `chain_ref` version performs reasonably well (though you again need to first bind the variable before performing the initial chain of modifications):

```rust
let mut foo = Foo::default();
foo.chain_ref().chain_ref().chain_ref();
if some_condition {
    foo.chain_ref().chain_ref().chain_ref();
}
consume_ref(&foo);
consume_move(foo);
```

> [Run in the Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2018&gist=80f4b820c323958a1a39c5bd0ae3e3ac)

Doing the same with `chain_move` can also be made to work, though it requires the value to be rebound *after* the conditional chain:

```rust
let foo = Foo::default();
let foo = if some_condition {
    foo.chain_move().chain_move().chain_move();
} else {
    foo
};
consume_ref(&foo);
consume_move(foo);
```

> [Run in the Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2018&gist=8b2ca33b23cbbabad739be513639001f)

Again, this is functional but somewhat awkward to construct (having the extra `else` branch only to satisfy the borrow checker) and not necessarily an obvious construction for someone who's not already familiar with the details of Rust's ownership rules.

In this case, both `chain_ref` and `chain_move` work, but both have ergonomic drawbacks as compared to the ideal version.

### Chaining Within a Function

Let's say you want to break some of your logic into a separate function. This is possible with both functions, thought the signature of your helper function will have to change depending on which chaining approach you are using:

```rust
fn do_modifications_ref(foo: &mut Foo) {
    foo.chain_ref().chain_ref().chain_ref();
}
```

```rust
fn do_modifications_move(foo: Foo) -> Foo {
    foo.chain_move().chain_move().chain_move()
}
```

It's worth noting that you can still use `chain_ref` within `do_modifications_move`, but you can't use `chain_move` within `do_modifications_ref` (since you can't take ownership of `foo`).

### No Chaining At All

Let's say you're a boring person and don't want to use method chaining at all, plain-old method calls are enough for you. If that's the case, the `chain_ref` version can also be used to modify the value without chaining, e.g.:

```rust
let mut foo = Foo::default();
foo.chain_ref();
foo.chain_ref();
foo.chain_ref();
```

The `chain_move` version can technically be used without chaining, but requires the variable to be re-bound in each statement, again making the code both harder to read and harder to write:

```rust
let foo = Foo::default();
let foo = foo.chain_move();
let foo = foo.chain_move();
let foo = foo.chain_move();
```

### A Real-World Example

In the abstract, this may seem like a number of minor issues and trivial complaints. To provide a real-world example of the implications these drawbacks have, let's look at an example that I ran into (one that motivated my writing this article).

Say you're writing a tool that uses [std::process::Command](https://doc.rust-lang.org/std/process/struct.Command.html) to spawn a child process. `Command` is designed to be used via method chaining by having all its methods take and then return `&mut self`. Your initial version looks something like this:

```rust
let result = Command::new("foo")
    .arg("--bar")
    .arg("--baz")
    .arg("quux")
    .status()
    .unwrap();
```

> [Run in the Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2018&gist=5124c4f3041bd78893e58fcdc815f0d6)

At some point later, you realize that you want to only pass the `--baz` flag conditionally, so you make the obvious changes to your code:

```rust
let command = Command::new("foo")
    .arg("--bar");

if set_baz {
    command.arg("--baz");
}

let result = command
    .arg("quux")
    .status()
    .unwrap();
```

> [Run in the Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2018&gist=179416e54846de7d564b465ff11b6e92)

But it doesn't compile, because you can't bind the result of the initial method chain when the chaining methods return `&mut Self` ([as `Command::arg` does](https://doc.rust-lang.org/std/process/struct.Command.html#method.arg)). To get it to work, you have to bind the result of `Command::new` to a variable, then perform all configuration on it:

```rust
let mut command = Command::new("foo");
command.arg("--bar");

if set_baz {
    command.arg("--baz");
}

let result = command
    .arg("quux")
    .status()
    .unwrap();
```

> [Run in the Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2018&gist=77df0254b3abde92ae409f259d545bec)

This is a minor bit of friction for someone already familiar with Rust, but it can be a frustrating (and unnecessary) roadblock for someone new to the language.

## Method Chaining is Orthogonal to Return Value

At this point, I feel comfortable in having demonstrated that returning `self` is, at best, an awkward way of implementing method chaining for a Rust type. Beyond that, though, it's worth asking a more fundamental question: *Should* it work better? Should we consider this a failing of Rust, that the language doesn't play well with method chaining? Or is there something fundamentally wrong with this form of method chaining?

To answer these questions, let's take a look at the function signature for `chain_ref`:

```rust
fn chain_ref(&mut self) -> &mut Self { ... }
```

Rust's type system allows us learn a lot about what a function can do solely based on its signature. Key here is that `chain_ref` only takes a single parameter: `&mut self`. We therefore know that it can (and almost certainly will) mutate `self` in some way. We also know that it is probably pure relative to `self`, such that the same value for `self` will produce the same mutation, since `chain_ref` takes no other parameters to influence its behavior.

But what does returning `&mut Self` tell us about `chain_ref`? Normally, the return type would tell us what the result of the operation is. But in this case, the returned value actually has nothing to do with the internal logic of `chain_ref`, it's only there to enable method chaining, which is completely orthogonal to `chain_ref` itself.

This becomes especially problematic if your function has an actual return value. Take [`HashMap::insert`](https://doc.rust-lang.org/std/collections/struct.HashMap.html#method.insert) as an example. `insert` returns the previous value if one was replaced, however it's not always necessary to check the return value. In some cases, I may want to insert many elements into a hash map, in which case using a method chain would be clear and concise:

```rust
let map = HashMap::new()
    .insert("foo", 1)
    .insert("bar", 2)
    .insert("baz", 3)
    .insert("quux", 4);
```

But there's no way to make this work while still returning a value from `insert`: You can either return a value or you can return `self`, but not both.

The fundamental problem with returning `self` solely for the purpose of enabling method chaining is that you're contorting your API in order to enable something that's completely orthogonal to what your API is doing. Your function's signature should reflect its behavior, and should be usable in a method chain regardless of its return type.

## We're Only Talking About Method Chains

At this point I've covered the practical issues with returning `self`  and the more conceptual reason why it doesn't make sense. Before I move on to discussing alternate solutions, I want to emphasize an important point: Returning `self` is only an issue if it's being done **solely to enable method chaining**. It's entirely reasonable to return `self` from a function if doing so is semantically meaningful, and I am in no way trying to say that it is never appropriate to return `self` from a method in Rust. It only becomes an issue if you're returning `self` from a function for no reason other than to allow users to chain those methods together.

## Method Cascades

As is often the case, we don't need to invent a whole new solution to this problem when we could simply steal good ideas from another programming language.

Dart provides first-class support for method chaining in the form of [method cascades](https://www.dartlang.org/guides/language/language-tour#cascade-notation-). The `..` operator is the the "cascaded method invocation operator", and behaves similarly to `.` except that discards the result of the method invocation and returns the original receiver instead. This allows *any* method to be chained in Dart, without requiring the author to have thought ahead of time to return `self`.

In Dart, the syntax looks something like this:

```dart
final addressBook = (AddressBookBuilder()
      ..name = 'jenny'
      ..email = 'jenny@example.com'
      ..phone = (PhoneNumberBuilder()
            ..number = '415-555-0100'
            ..label = 'home')
          .build())
    .build();
```

While there's no equivalent syntax built into Rust, we could achieve something very similar with the help of a fairly simple macro. In fact, there's already the [cascade](https://crates.io/crates/cascade) crate which does just that!

```rust
let foo = cascade! {
    foo: Foo::default();
    ..chain();
    ..chain();
    ..chain();
    | if some_condition {
        cascade! {
            &mut foo;
            ..chain();
            ..chain();
            ..chain();
        }
    };
    ..chain();
    ..chain();
    ..chain();
};
consume_ref(&mut foo);
consume_move(foo);
```

Since this pattern hasn't yet seen wide usage in the Rust community, I expect that it will take some time and iteration to fully adapt it to Rust as a language (though the cascade crate is certainly a good start). Looking to the future, I would personally like to see this pattern become "official" in some regards, either through inclusion in the standard library or 🤞 a native syntax 🤞.

## Conclusion

So in summary:

* Don't return `self` from methods if you're **only** doing so to enable chaining.
* Start using the `cascade` crate instead!

[builder pattern]: https://github.com/rust-unofficial/patterns/blob/master/patterns/builder.md
