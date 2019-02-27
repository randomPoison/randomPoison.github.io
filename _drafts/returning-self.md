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

While the above example demonstrates the most straightforward way of doing method chaining (i.e. initializing and modyfing an object in a single statement), there are oftem more complex use cases that don't work nearly as well.

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

The most basic case is having a single long method chain, from construction into the consumtion of your type:

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

In this case, the `chain_ref` version performs reasonably well (though you again need to first bind the variable before performing the inital chain of modifications):

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

The `move_self` version again requires the variable to be re-bound in each statement, again having a negative impacts on the ergonomics of the code:

```rust
let foo = Foo::default();
let foo = foo.chain_move();
let foo = foo.chain_move();
let foo = foo.chain_move();
```

Results Summary
---------------

Results for `chain_move`:

|                    | `consume_move` | `consume_ref` |
|--------------------|----------------|---------------|
| Single chain       | yes            | yes           |
| Use before consume | yes            | yes           |
| Modify bound value | no             | no            |
| In a function      | not ergonomic  | no            |
| No chaining at all | not ergonomic  | not ergonomic |

Results for `chain_ref`:

|                    | `consume_move` | `consume_ref` |
|--------------------|----------------|---------------|
| Single chain       | no             | yes           |
| Use before consume | no             | no            |
| Modify bound value | yes            | yes           |
| In a function      | yes            | yes           |
| No chaining at all | yes            | yes           |

Conclusions
-----------

So... what? What's the point of all of this? I can imagine that some folks reading this are going to think the examples I've chosen are bad or overexaggerate the ergonomic issues with some of the approaches, or that they're too contrived and don't represent realistic use cases.

While I don't expect that I can convince everyone that the examples I've chosen are valid, suffice it to say that these are all cases that I have run into personally. With both methods that return `Self` and ones that return `&mut Self`, in various projects, I've run into cases where the clean, obvious thing I wanted to do wasn't possible (or wasn't ergonomic) due to one of the issues demonstrated above. In fact, the whole reason I've bothered to write this overly-long analysis in the first place was that I've run into this issue so many times that I wanted to demonstrate clearly and thoroughly that this is, in fact, a problem!

To provide a real-world example, though: Let's say you're writing a tool that uses [std::process::Command](https://doc.rust-lang.org/std/process/struct.Command.html) to spawn a child process. Your initial version looks something like this:

```rust
let result = Command::new("foo")
    .arg("--bar")
    .arg("--baz")
    .arg("quux")
    .status()
    .unwrap();
```

At some point later, you realize that you want to only pass `--baz` flag conditionally, so you make the obvious changes to your code:

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

And it doesn't compile, because you can't bind the result of the initial method chain when the chaining methods return `&mut Self` ([as `Command::arg` does](https://doc.rust-lang.org/std/process/struct.Command.html#method.arg)).

This is something that's easy for me to mess up as a fairly experienced Rust developer, and it can be absolutely confusing and frustrating for those new to Rust.

---

Extra references and stuff:

* [Template gist](https://gist.github.com/6aa2a0992ed5043e72ed804e5f221101)
* [Complete demo](https://play.rust-lang.org/?version=stable&mode=debug&edition=2018&gist=1bf8a2fe6044498841aeabb223fab6f9)

[builder pattern]: https://github.com/rust-unofficial/patterns/blob/master/patterns/builder.md
