---
layout: post
comments: true
title: Rust Todo SQL Example Application
excerpt_separator: <!--more-->
---

Coming across "[Top 15 Rust Projects To Elevate Your Skills](https://zerotomastery.io/blog/rust-practice-projects/)", I decided to build a Todo CLI application (_quite the road less traveled_, eh?) and defined a few goals/requirements after checking some of their suggested crates and examples:

- Use a database to persist todos (I used SQLite).
- Minimize the use of `unwrap()` and `expect()` calls.
- Include relevant working test cases, even if they are not 100% necessary.

![](https://github.com/dezoito/dezoito.github.io/blob/dev/public/images/todo-demo.gif?raw=true)

This article is a walk through the process of building the app and some of the choices made.

The resulting source code is available at [https://github.com/dezoito/rust-todo-list](https://github.com/dezoito/rust-todo-list).

# Table of Contents

- [Part 1: Getting Started](#part-1-getting-started)
- [Part 2: Struct and Database Definitions](#part-2-struct-and-database-definitions)
- [Part 3: Adding Functionality](#part-3-adding-functionality)
- [Part 3: Testing the Application](#part-3-testing-the-application)

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

---

<p>&nbsp;</p>

## Part 1: Getting started

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

#[allow(unused)] // Remove later
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

Is the user types `todo add Write a Tutorial`, `add` would be the command and `"Write a tutorial"` would be the suffix.

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

FUll code for this section can be seen at [https://github.com/dezoito/rust-todo-list/tree/part1](https://github.com/dezoito/rust-todo-list/tree/part1)

In Part 2 we'll define the properties of a todo entry, and see how we can wire up the SQLite database.

---

<p>&nbsp;</p>

## Part 2: Struct and Database Definitions

We need to define the properties of a `todo` entry first, and one way to do that is by defining a `struct`:

```rs
#[derive(Debug)]
pub struct Todo {
    pub id: i32,
    pub name: String,
    pub date_added: String,
    pub is_done: i32,
}

```

To make matching the datatypes available in SQLite (which lacks date-time and boolan fields), `data_added` is defined as string, and `is_done` is defined as an integer.

We also need a way to:

1. Check if we have a database file where it's expected and create one otherwise;

2. Connect to the database;

3. Check if it has the expected table or create it;

4. Return a database conncection reference that we can use in other parts of the code;

We solve this by adding the following code to `lib.rs`

```rs

use console::style;
use rusqlite::{Connection, Result};
use std::env;
use std::fs;
use std::path::Path;
use std::path::PathBuf;

...

// Get the user's home directory whether they use Linux, MacOS or Windows
fn get_home() -> String {
    let home_dir = match env::var("HOME") {
        Ok(path) => PathBuf::from(path),
        Err(_) => {
            // Fallback for Windows and macOS
            if cfg!(target_os = "windows") {
                if let Some(userprofile) = env::var("USERPROFILE").ok() {
                    PathBuf::from(userprofile)
                } else if let Some(homedrive) = env::var("HOMEDRIVE").ok() {
                    let homepath = env::var("HOMEPATH").unwrap_or("".to_string());
                    PathBuf::from(format!("{}{}", homedrive, homepath))
                } else {
                    panic!("Could not determine the user's home directory.");
                }
            } else if cfg!(target_os = "macos") {
                let home = env::var("HOME").unwrap_or("".to_string());
                PathBuf::from(home)
            } else {
                panic!("Could not determine the user's home directory.");
            }
        }
    };

    // Convert the PathBuf to a &str
    match home_dir.to_str() {
        Some(home_str) => home_str.to_string(),
        None => panic!("Failed to convert home directory to a string."),
    }
}

// Aux function that creates the folder where the DB should be stored
// if it doesn't exist
pub fn verify_db_path(db_folder: &str) -> Result<()> {
    if !Path::new(db_folder).exists() {
        // Check if the folder doesn't exist
        match fs::create_dir(db_folder) {
            Ok(_) => println!("Folder '{}' created.", db_folder),
            Err(e) => eprintln!("Error creating folder: {}", e),
        }
    }
    Ok(())
}

// Aux function that creates tables if they don't exist
pub fn verify_db(conn: &Connection) -> Result<()> {
    conn.execute(
        "CREATE TABLE IF NOT EXISTS todo (
    	id	        INTEGER NOT NULL,
    	name	    TEXT NOT NULL,
    	date_added	REAL NOT NULL DEFAULT current_timestamp,
    	is_done	    NUMERIC NOT NULL DEFAULT 0,
    	    PRIMARY KEY(id AUTOINCREMENT)
    )",
        [], // no params for this query
    )?;
    Ok(())
}


// Returns a connection, creating the database if needed
pub fn get_connection() -> Result<Connection> {
    let db_folder = get_home() + "/" + "todo_db/";
    let db_file_path = db_folder.clone() + "todo.sqlite";
    verify_db_path(&db_folder)?;
    let conn = Connection::open(db_file_path)?;
    verify_db(&conn)?;
    Ok(conn)
}

```

The `verify_db` function is also interesting in that it shows the SQL table definitions for `todos` and the datatypes used for that particular DB flavour.

It's worth noting that most of these functions returns a `Result<>` (in this case defined in `rusqlite` crate), so they'll either propagate an error to the caller, or return an unwrapped value.

We need to update the code in `main.rs` to expect that return type as well:

```rs

fn main() -> Result<()> {
    let args: Vec<String> = env::args().collect();

    // Get a connection to the DB
    let conn = get_connection()?;

    ...

    match command.as_str() {
        ...
    };

    Ok(())
}

```

Notice we added a call to `get_connection()` and an `Ok(())` after the match statement to satisfy the `Result<()>` return type.

Commited code up to this point can be seen at [https://github.com/dezoito/rust-todo-list/tree/part2](https://github.com/dezoito/rust-todo-list/tree/part2).

---

<p>&nbsp;</p>

## Part 3: Adding Functionality

---

<p>&nbsp;</p>

## Part 3: Testing the Application
