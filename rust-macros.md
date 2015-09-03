title: Macros in Rust
author:
  name: Ivan Smirnov
  twitter: aldanor
  url: http://ivansmirnov.io
output: rust-macros.html
style: assets/style.css
controls: true

--

# Macros in Rust
## Rust Meetup, Dublin / Sep 3, 2015 / Ivan Smirnov

--

### Things to commit before leaving your job

```cpp
#define struct union
#define if while
#define else
#define break
#define double float
#define volatile
#define false true
#define sizeof(x) (sizeof(x) - 1)
#define if(x) if ((x) && ((__LINE__ & 15) != 0))
```

![](assets/things-to-commit.jpg)

--

### Macros in Rust: an overview

- Pattern-matching macro system based on *Macro-by-Example*<sup>1</sup>
- *Macro hygiene*: macro expansion is guaranteed not to cause accidental capture
  of identifiers
- Macro expansion has its own compiler phase; it is not a full pass over
  the source code
- Macro invocations can only appear where they are explicitly supported:
  items, methods, statements, expressions, patterns
- There are limitations: e.g., a macro cannot generate identifier for a
  function declaration (there's `concat_idents!` but it's behind a feature gate)
- The outermost macro invocation is expanded first

<small><sup>1</sup> E. Kohlbecker, M. Wand (1986). Macro-by-Example: Deriving Syntactic Transformations from their Specifications (1986).
https://www.cs.indiana.edu/ftp/techreports/TR206.pdf</small>

--

### Macro hygiene in action

```rust
macro_rules! print_x {
    () => { println!("{}", x) }
}

fn main() {
    let x = 1;
    print_x!();
}
```

```
test.rs:2:28: 2:29 error: unresolved name `x`
test.rs:2     () => { println!("{}", x) }
```

![](assets/macro-hygiene.jpg)

--

### Hello, world!

The very first example in the official Rust book actually contains a macro:

```rust
fn main() {
    println!("{}", "Hello, world!");
}
```

Here, `println!` is not a function – it's a macro that gets expanded at *compile time*
and is defined in the standard library like so:

```rust
macro_rules! println {
    ($fmt:expr) => (print!(concat!($fmt, "\n")));
    ($fmt:expr, $($arg:tt)*) => (print!(concat!($fmt, "\n"), $($arg)*));
}
```

which expands to:

```rust
fn main() {
    ::std::io::_print(format_args!("{}", "Hello, world!"));
}
```

Format strings are type-safe and *statically* verified:

```rust
println!("{} {} {}", 2, "foo"); // compile-time error, missing argument
```

--

### Using macros

Macro invocation looks a lot like a function call, but:

- macro name must be followed by an exclamation mark, `!`
- parentheses, square brackets and curly braces are ok: `()`, `[]`, `{}`
- content inside the parentheses doesn't have to be valid Rust
- ... as long as the parentheses are balanced (parsed into token tree)

Either way works:

```rust
println!("{}", "foo");
println!["{}", "foo"];
println!{"{}", "foo"};
```

An example where it's common to use square brackets:

```rust
let v = vec![1, 2, 3];
```

which expands to:

```rust
let v = <[_]>::into_vec(Box::new([1, 2, 3]));
```

--

### Defining macros

A simplest macro definition: macro with no arguments.

```rust
macro_rules! say_hello {
    () => (
        println!("Hello");
    )
}

fn main() {
    say_hello!();
}
```

Can also use square brackets/braces and drop the semicolon:

```rust
macro_rules! say_hello {
    [] => {
        println!("Hello")
    }
}
```

--

### Passing arguments to a macro

```rust
macro_rules! say {
    ($what:expr) => (
        println!("{}", $what)
    )
}

fn main() {
    say!("Hello");
    say!(1 + 2);
}
```

In this macro, `expr` is a *designator*.

There's a whole bunch of other designators, most commonly used ones being
`ty`, `ident` and `block`.

--

### More designators and `stringify!`

```rust
macro_rules! define {
    ($name:ident[$ty:ty]($lhs:ident, $rhs:ident) => $body:expr) => (
        fn $name($lhs: $ty, $rhs:$ty) -> $ty {
            let result = $body;
            println!("Given {} = {}: {} and {} = {}: {}",
                     stringify!($lhs), $lhs, stringify!($ty),
                     stringify!($rhs), $rhs, stringify!($ty));
            println!("{}({}, {}) = {} = {}",
                     stringify!($name), stringify!($lhs), stringify!($rhs),
                     stringify!($body), result);
            result
        }
    )
}

fn main() {
    define!(add_double[i32](x, y) => 2 * x + 2 * y);
    println!("Answer: {}", add(5, 16));
}
```

The output:

```
Given x = 5: i32 and y = 32: i32
add_double(x, y) = 2 * x + y = 42
Answer: 42
```

`stringify!` captures macro arguments as literal strings, exactly as they are passed
at the call site, and is very useful for debugging.

--

### Blocks are expressions too

Whenever an `expr` designator is expected, you can always pass a block:

```rust
fn main() {
    define!(add[f64](a, b) => {
        let c = a * b;
        a + b + c
    });
}
```

However, `block` designator doesn't allow non-block expressions:

```rust
macro_rules! block_macro {
    ($block:block) => { $block }
}

fn main() {
    let (x, y) = (1, 2);

/*
    block_macro!(x + y);
    // test.rs:7:10: 7:11 error: expected `{`, found `x`
    // test.rs:7     foo!(x + y);
*/

    println!("{}", block_macro!({ x + y }));
}
```

```
3
```

--

### Macro rule overloads

Let's modify `define!` macro so it can define unary operators as well:

```rust
macro_rules! define {
    ($name:ident[$ty:ty]($lhs:ident, $rhs:ident) => $body:expr) => (
        fn $name($lhs: $ty, $rhs:$ty) -> $ty { $body }
    );

    ($name:ident[$ty:ty]($arg:ident) => $body:expr) => (
        fn $name($arg: $ty) -> $ty { $body }
    )
}

fn main() {
    define!(add[i32](x, y) => x + y);
    define!(neg[i32](x) => -x);
    println!("{}, {}", add(3, 4), neg(5));
}
```

```
7, -5
```

Note: each arm must be separated with a semicolon.

--

### Resolving multiple matches

Macro rules are matched in the order specified until a match is found.

```rust
macro_rules! multi_arm {
    ($x:expr) => (
        println!("arm 1")
    );
    ($x:expr) => (
        println!("arm 2")
    );
}

fn main() {
    multi_arm!(1);
}
```

Although both arms can be matched, only the first one is evaluated, and
so the output is:

```
arm 1
```

--

### Repeating arguments

Can `define!` generate functions with any number of parameters?

```rust
macro_rules! define {
    ($name:ident[$ty:ty]($($arg:ident);*) => $body:expr) => (
        fn $name($($arg: $ty),*) -> $ty { $body }
    );
}

fn main() {
    define!(add[i32](x; y) => x + y);
    define!(neg[i32](x) => -x);
    define!(zero[i32]() => 0);
    define!(volume[i32](w; d; h) => w * d * h);
    println!("{}, {}, {}, {}", add(3, 4), neg(5), zero(), volume(2, 3, 4));
}
```

Note that the list of function arguments is now semicolon-separated.

Defining repeated arguments:

* `$(...),*` matches the argument zero or more times
* `$(...),+` matches the argument one or more times

---

### Recursive macros

Let's write a macro that counts the expression arguments passed to it
(statically, so that it's result is a compile-time known const).

```rust
macro_rules! count_exprs {
    () => { 0 };
    ($e:expr) => { 1 };
    ($head:expr, $($tail:expr),+) => { 1 + count_exprs!($($tail),*) };
}

fn main() {
    const COUNT: usize = count_exprs!(1 + 2, "foo", 42);
    println!("{}", COUNT);
}

```

```
3
```

Macro call in this example would get expanded to

```rust
const COUNT: usize = 1 + 1 + 1;
```

which the compiler will then fold into just

```rust
const COUNT: usize = 3;
```

--

### Macro-related attributes

`#[macro_use]` attribute allows loading macros from other crates:

```rust
#[macro_use]
extern crate foo;
```

Only import a limited set of macros:

```rust
#[macro_use(foo, bar)]
extern crate baz;
```

This can be also used directly on a module:

```rust
#[macro_use]
mod {
    // macros defined here will be visible in module's parent
}
```

Export a macro for cross-crate usage:

```rust
#[macro_export]
macro_rules! foo {
    // this macro definition will be available to other crates, and will
    // be imported automatically by #[macro_use] without arguments
}
```

--

### The order matters

If module `foo` uses a macro from module `macros`, then the following wouldn't work:

```rust
// lib.rs

mod foo;
#[macro_use]
mod macros;
```

but this would:

```rust
// lib.rs

#[macro_use]
mod macros;
mod foo;
```

--

### `$crate` metavar

A macro can reference various functions in the crate directly:

```rust
// crate foo

#[macro_export]
macro_rules! some_macro {
    () => { ::foo::some_module::some_function() }
}
```

But what if this macro was imported in another crate like so?

```rust
#[macro_use] extern crate "foo" as bar;
```

`$crate` metavar is the recommended way of handling this:

```rust
#[macro_export]
macro_rules! some_macro {
    () => { $crate::some_module::some_function() }
}
```

Note: macro-importing `extern crate` can only appear at the crate root.

--

### Standard library: `try!`

```rust
macro_rules! try {
    ($expr:expr) => (match $expr {
        $crate::result::Result::Ok(val) => val,
        $crate::result::Result::Err(err) => {
            return $crate::result::Result::Err($crate::convert::From::from(err))
        }
    })
}
```

Typical error-handling logic:

```rust
let result = match inner() {
    Err(err) => return Err(From::from(err)),
    Ok(val) => match outer(val) {
        Err(err) => return Err(From::from(err)),
        Ok(val) => {
            // ...
        }
    }
};
Ok(process(result));
```

With `try!` macro this becomes just

```rust
let result = try!(outer(try!(inner()));
Ok(process(result));
```

--

### Standard library: inspection macros

It is possible to get the current module path, file, line number and column number
via standard library macros:

```rust
macro_rules! inspect {
    () => {
        println!("{} in {}:{}:{}", module_path!(), file!(), line!(), column!())
    }
}

pub mod mod1 {
    pub mod mod2 {
        pub fn foo() {
            inspect!();
        }
    }
}

fn main() {
    mod1::mod2::foo();
}
```

The output is:

```
test::mod1::mod2 in test.rs:10:12
```

--

<!--
TODO:

* examples: lazy_static! etc
* std lib examples: defining tuple impls recursively
* hdf5-rs examples
 -->
