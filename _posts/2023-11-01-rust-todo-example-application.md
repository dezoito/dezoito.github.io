---
layout: post
comments: true
title: Rust Todo SQL Example Application - Part 1
excerpt_separator: <!--more-->
---

Coming across "[Top 15 Rust Projects To Elevate Your Skills](https://zerotomastery.io/blog/rust-practice-projects/)", I decided to build a Todo CLI application (_quite the road less traveled_, eh?) and defined a few goals/requirements after checking some of their suggested crates and examples:

- Use a database to persist todos (I used SQLite).
- Minimize the use of `unwrap()` and `expect()` calls.
- Include relevant working test cases, even if they are not 100% necessary.

![](https://github.com/dezoito/dezoito.github.io/blob/dev/public/images/todo-demo.gif?raw=true)

This article is a walk through the process of building the app and some of the choices made.

The resulting source code is available at [https://github.com/dezoito/rust-todo-list](https://github.com/dezoito/rust-todo-list).

> > > Article TOC here <<<<

<!--more-->

### Functional Requirements

I wanted the user to be able to interact with the database by running simple CLI commands - inspired by this [CLI todo app](https://github.com/sioodmy/todo) - at least for this first iteration, and settled for the following API:

```
    - add [TASK]
        Ads new task/s
        Example: todo add "Build a tree"

    - list
        Lists all tasks
        Example: todo list

    - toggle [ID]
        Toggles the status of a task (Done/Pending)
        Example: todo toggle 2

    - rm [ID]
        Removes a task
        Example: todo rm 4

    - sort
        Sorts completed and pending tasks

    - reset
        Deletes all tasks

```

### Database

At first I was going to use a key/value store called [SLED](https://crates.io/crates/sled) to persist data, but found out that it is more suited for long running connections (such as the ones from a web based API), then pivoted to using SQLite via the [Rusqlite](https://crates.io/crates/rusqlite) crate.

Turns out that was a great choice for the scope of a first project, as all your data can be stored in a single file and you can even use an "in memory" database when running tests with very little configuration.

## Gettin started

Init the Rust project by running

```sh
cargo init todo
```

The add the following dependencies and definitions to the generated `Cargo.toml` file:

```toml
[package]
name = "todo"
version = "0.1.0"
edition = "2021"

[dependencies]
console = "0.15.7"
dialoguer = "0.11.0"
lazy_static = "1.4.0"
rusqlite = { version = "0.29.0", features = ["bundled"] }
```

### Code structure.

Keeping it simple, we have two Rust files in the `src/` folder:

- `main.rs`: Entry point that accepts arguments and routes execution to functions accordingly.

- `lib.rs`: Contains the functions needed to connect and interact with the database and print the records. Will also contain tests.

The first iteration of the code looks like this:

`src/main.rs`

```rs
extern crate todo;
use std::env;
use todo::*;

fn main() {
    let args: Vec<String> = env::args().collect();

    if args.len() == 1 {
        println!("You need to pass some arguments!");
        help();
        std::process::exit(1);
    }

    let command = &args[1];
    let suffix = &args[2..].iter().cloned().collect::<Vec<_>>().join(" ");

    match command.as_str() {
        "add" => {}
        "list" => {}
        "toggle" => {}
        "reset" => {}
        "rm" => {}
        "sort" => {}
        "help" | "--help" | "-h" | _ => help(),
    };
}

```

In summary, we parse the command line arguments into `command` and `suffix` variables.

Is the user types `todo add Write a Tutorial`, `add` would be the command and `Write a tutorial` would be the suffix.

We then match the `command` to the corresponding function, and if there's no match we call `help()` from our `lib.rs` file below:

`src/lib.rs`

```rs
use console::style;

// Prints help with a list of commands and parameters
pub fn help() {
    let help_title = "\nAvailable commands:";
    let help_text = r#"
        - add [TASK]
            Ads new task/s
            Example: todo add "Build a tree"

        - list
            Lists all tasks
            Example: todo list

        - toggle [ID]
            Toggles the status of a task (Done/Pending)
            Example: todo toggle 2

        - rm [ID]
            Removes a task
            Example: todo rm 4

        - sort
            Sorts completed and uncompleted tasks

        - reset
            Deletes all tasks
        "#;

    println!("{}", style(help_title).cyan().bright());
    println!("{}", style(help_text).green());
}

```

Right now we are only implementing the `help()` function and using the `console` crate to colorize the output.

We can check if this works by running:

```sh
cargo run help
```

Or better yet, leaving a console window open and running

```sh
cargo watch -c -x "run -- help"
```

The last option reloads the app every time we make changes to our code.

---

In Part 2 we'll define the properties of a todo entry, and see how we can wire up the SQLite database.
