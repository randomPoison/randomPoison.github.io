---
layout: post
title: "Returning Self Considered An Anti-Pattern"
categories: rust
---

<!-- markdownlint-disable MD002 -->

It's a common pattern in the Rust ecosystem to have a function return some variation of `Self` (either `Self`, `&Self`, or `&mut Self`) i<!-- markdownlint-disable MD037 -->n order to enable method chaining. For example:

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

While the above example demonstrates the most straightforward way of doing method chaining (i.e. initializing and modifying an object in a single statement), there are often more complex use cases that don't work nearly as well.

To demonstrate this, we're going to work with the following definitions:

```rust
#[derive(Debug, Default)]
struct Foo {
    value: usize,
}

impl Foo {
    fn chain_move(mut self) -> Self {
        self.value += 1;
        self
    }

    fn chain_ref(&mut self) -> &mut Self {
        self.value += 1;
        self
    }
}

fn consume_move(foo: Foo) {
    println!("{:?}", foo);
}

fn consume_ref(foo: &Foo) {
    println!("{:?}", foo);
}
```

This allows us to compare the two primary ways that people implement method chaining: Returning `Self`, and returning `&mut Self`. We also look at two ways of consuming the value: By ref and by value. As we'll see, the way the final object will be consumed often interacts differently with the different chaining methods.

Let's now take a look at each of the use cases we would like to support, and see how they work with each of the method chaining approaches.

Single Method Chain
-------------------

The most basic case is having a single long method chain, from construction into the consumption of your type:

```rust
consume_move(Foo::default().chain_move().chain_move());
consume_ref(Foo::default().chain_ref().chain_ref());
```

> [Run in the Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2018&gist=9402739e03bf725420ad70a0283ac943)

For both the ref and move versions, a single large chain works as expected, and you can pass the output directly into the consuming function. Note, though, that while the `chain_move` version works with both `consume_move` and `consume_ref` version, `chain_ref` can only be used with `consume_ref`. If we try to pass the result of `chain_ref` into `consume_move`, we get this error:

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

Summary:

|              | `consume_move` | `consume_ref`|
|--------------|----------------|--------------|
| `chain_move` | yes            | yes          |
| `chain_ref`  | no             | yes          |

Use Before Consuming
--------------------

Let's say you needed to log the value of `Foo` before consuming it. This works fine with the `chain_move`:

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

As the compiler helpfully notes, the temporary value created by `Foo::default()` is dropped at the end of the chain expression, so we can't bind it to a variable. Instead, we have to split binding the variable and performing the method chain into separate expressions:

```rust
let mut foo = Foo::default();
foo.chain_ref().chain_ref();
println!("foo: {:?}", foo);
consume_ref(&foo);
consume_move(foo);
```

> [Run in the Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2018&gist=234824151a664395c788cac897c49a9e)

While this is functional, it's worth noting that it has a few drawbacks as compared to the `chain_move` version:

* You can no longer create and modify the value in a single expression.
* You have to bind `foo` as a mutable variable, which loosens some of the guarantees you get in the `chain_move` version when binding the variable immutably.
* When converting from the single chain version to this version, it's easy to initially apply the naive transformation shown above and get tripped up when it doesn't work. The `chain_move` version, on the other hand, works fine with the naive transformation.

For this case, both `chain_ref` and `chain_move` work equally well with `consume_ref` and `consume_move` since, once the object is bound to a variable, it is easy to either lend that value to another function or to transfer ownership entirely.

Summary:

|              | `consume_move` | `consume_ref` |
|--------------|----------------|---------------|
| `chain_move` | yes            | yes           |
| `chain_ref`  | not ergonomic  | not ergonomic |

Modifying an Owned Value
-----------------------

Now let's say that you want want perform and initial method chain, then conditionally apply another chain of operations to the same object. This means that we already have a bound, mutable variable that we would like to modify in the same method-chaining style that we use to crate the object.

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

Again, this is functional but somewhat awkward to construct (having the unnecessary `else` branch only to satisfy the borrow checker) and not necessarily an obvious construction for someone who's not already familiar with the details of Rust's ownership rules.

Summary:

|              | `consume_move` | `consume_ref` |
|--------------|----------------|---------------|
| `chain_move` | not ergonomic  | not ergonomic |
| `chain_ref`  | yes            | yes           |

Chaining Within a Function
--------------------------

Let's say you want to break some of your logic into a separate function. Again, this is possible with both approaches but a bit less ergonomic/idiomatic with `chain_move`:

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

It's worth noting that you can still use `chain_ref` within `do_modifications_move`, but you can't use `chain_move` within `do_modifications_ref`.

No Chaining At All
------------------

Let's say you're a boring person and don't want to use method chaining at all. Plain-old method calls are enough for you. If that's the case, the `move_ref` version can also be used to modify the value without chaining, e.g.:

```rust
let mut foo = Foo::default();
foo.chain_ref();
foo.chain_ref();
foo.chain_ref();
```

The `move_self` version again requires the variable to be re-bound in each statement, again making the code both harder to read and harder to write:

```rust
let foo = Foo::default();
let foo = foo.chain_move();
let foo = foo.chain_move();
let foo = foo.chain_move();
```

A Real-World Example
--------------------

To provide a real-world example, though: Let's say you're writing a tool that uses [std::process::Command](https://doc.rust-lang.org/std/process/struct.Command.html) (which is designed to be used via method chaining by having all its methods return `&mut Self`) to spawn a child process. Your initial version looks something like this:

```rust
let result = Command::new("foo")
    .arg("--bar")
    .arg("--baz")
    .arg("quux")
    .status()
    .unwrap();
```

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

And it doesn't compile, because you can't bind the result of the initial method chain when the chaining methods return `&mut Self` ([as `Command::arg` does](https://doc.rust-lang.org/std/process/struct.Command.html#method.arg)). To get it to work, you have to bind the result of `Command::new` to a variable, then perform all configuration on it:

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

This is something that's easy for me to mess up as a fairly experienced Rust developer, and it can be absolutely confusing and frustrating for those new to Rust. On a more subjective note, the resulting code is also pretty gnarly and loses a lot of the simplicity and readability that the original version had.

It's Ultimately a Hack
----------------------

All I've done so far is demonstrate that returning `Self` to enable method chaining *doesn't* work. In order to truly explain why doing so should be considered an anti-pattern, I need to answer a more fundamental question: *Should* it work? Should we consider this a failing of Rust, that it doesn't play well with method chaining? Or is there something fundamentally wrong with this form of method chaining?

To examine this, let's take a look at the function signature for `chain_ref`:

```rust
fn chain_ref(&mut self) -> &mut Self { ... }
```

Rust's strong type system allows us learn a lot about what a function can do solely based on its signature. Key here is that `chain_ref` only takes a single parameter: `&mut self`. We therefore know that it can (and almost certainly will) mutate `self` in some way. We also know that it must be pure relative to `self`, such that the same value for `self` will produce the same mutation, since `chain_ref` takes no other parameters to influence its behavior.

But what does returning `&mut Self` tell us about `chain_ref`? Normally, the return type would tell us what the result of the operation is. But in this case, the returned value actually has nothing to do with the internal logic of `chain_ref`, it's only there to enable method chaining, which is completely orthogonal to `chain_ref` itself.

This becomes especially problematic if your function has an actual return value. Take [`HashMap::insert`](https://doc.rust-lang.org/std/collections/struct.HashMap.html#method.insert) as an example. `insert` returns the previous value if one was replaced, however it's not always necessary to check the return value, so it should be fine to use it in a method chain:

```rust
let map = HashMap::now()
    .insert("foo", 1)
    .insert("bar", 2)
    .insert("baz", 3)
    .insert("quux", 4);
```

But there's no way to make this work while still returning a value from `insert`.

This is why, in my mind, returning `Self` solely to enable method chaining is a hack and an anti-pattern: You're contorting your API in order to enable something that's completely orthogonal to what your API is doing. In the best case scenario it harmless if inconvenient. In the worst case it can actively make some uses cases impossible without adding non-chaining method alternatives.

Cascading as a Better Alternative
---------------------------------

Now for the denouement, the part where I reveal the grand solution to all the problems I have laid out. As is often the case with Rust, we don't need to invent a whole new solution to this problem when we could simply steal the solution from another programming language. In this case it's Smalltalk as channelled through Dart.

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

Looking to the Future
---------------------

I think `cascade` is great and sufficient to allow us to stop returning `Self` for the sole purpose of enabling method chaining, but I think we'll ultimately want a syntax built into the language to enable method chaining. The cascade syntax in Dart is pure syntactic sugar, which is all method chaining should be: A more convenient way of calling many methods without having any impact on the actual functionality.

---

Extra references and stuff:

* [Template gist](https://gist.github.com/6aa2a0992ed5043e72ed804e5f221101)
* [Complete demo](https://play.rust-lang.org/?version=stable&mode=debug&edition=2018&gist=1bf8a2fe6044498841aeabb223fab6f9)

`Command` example using cascade:

```rust
let result = cascade! {
    Command::new("foo");
    ..arg("--bar");
    ..arg("--baz");
    ..arg("quux");
}.status().unwrap();

// Becomes:

let result = cascade! {
    command: Command::new("foo");
    ..arg("--bar");
    | if set_baz {
        command.arg("--baz");
    };
    ..arg("quux");
}.status().unwrap();
```

[builder pattern]: https://github.com/rust-unofficial/patterns/blob/master/patterns/builder.md
