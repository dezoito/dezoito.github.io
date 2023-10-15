---
layout: post
comments: true
title: Expressiveness in Rust code
excerpt_separator: <!--more-->
---

After a quick (and failed) attempt to build a desktop application using [Tauri](https://tauri.app/), I decided to learn Rust and started by using the excellent [Rustlings](https://github.com/rust-lang/rustlings) resource.

<!--more-->

I like to compare my solutions and experiment with different ways to solve the challenges, and I've documented that in my commented solutions repository:

[https://github.com/dezoito/rustlings-commented-solutions](https://github.com/dezoito/rustlings-commented-solutions)

The first time I saw the language, I had the impression that it was excessively verbose, but the brief experience has changed my mind – to some extent.

For example, in `exercises/conversions/try_from_into.rs`, we are challenged to implement different conversion methods for a `Color` struct:

### Conversion from Tuple to Color Struct

Without getting into too much detail, the second version uses variable destructuring and Rust's type system to validate the data:

```rust

// My horribly verobose version:
fn try_from(tuple: (i16, i16, i16)) -> Result<Self, Self::Error> {
    let is_valid_range = |x: i16| (0..=255).contains(&x);

    if is_valid_range(tuple.0) && is_valid_range(tuple.1) && is_valid_range(tuple.2) {
        Ok(Color {
            red: tuple.0 as u8,
            green: tuple.1 as u8,
            blue: tuple.2 as u8,
        })
    } else {
        Err(IntoColorError::IntConversion)
    }
}
```

```rust
// A more idiomatic approach:
fn try_from(tuple: (i16, i16, i16)) -> Result<Self, Self::Error> {
    let (r, g, b) = tuple;
    let (red, green, blue) = (
        u8::try_from(r).or(Err(IntoColorError::IntConversion))?,
        u8::try_from(g).or(Err(IntoColorError::IntConversion))?,
        u8::try_from(b).or(Err(IntoColorError::IntConversion))?,
    );
    Ok(Color { red, green, blue })
}
```

The second version is shorter, more readable, but wait, things get better...

### Conversion from an Array to Color Struct

```rust
// I repeat the logic for the tuple conversion
fn try_from(arr: [i16; 3]) -> Result<Self, Self::Error> {
    let is_valid_range = |x: i16| (0..=255).contains(&x);

    if is_valid_range(arr[0]) && is_valid_range(arr[1]) && is_valid_range(arr[2]) {
        Ok(Color {
            red: arr[0] as u8,
            green: arr[1] as u8,
            blue: arr[2] as u8,
        })
    } else {
        Err(IntoColorError::IntConversion)
    }
}

```

```rust
// A smarter person just converts the array into a tuple and
// uses the previous implementation
fn try_from(arr: [i16; 3]) -> Result<Self, Self::Error> {
    let [r, g, b] = arr;
    (r, g, b).try_into()
}

```

### Conversion from an Array Slice to Color Struct

A similar pattern can be used in the other implementations (I'm omitting mine since it looks as convoluted as the previous):

```rust
// Attempts to convert from a slice, returns appropriate error
// if it fails
fn try_from(slice: &[i16]) -> Result<Self, Self::Error> {
    match slice {
        [r, g, b] => Color::try_from((*r, *g, *b)),
        _ => Err(IntoColorError::BadLen)?
    }
}

```

In summary, my brief experience with Rust has taught me that while it may initially seem verbose, the language's true expressiveness becomes evident when you invest the time to learn it.
