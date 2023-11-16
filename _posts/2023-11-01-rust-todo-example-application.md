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
- [Part 4: Testing the Application](#part-4-testing-the-application)
- [Part 5: Building the Executable](#part-5-building-the-executable)

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

We solve this by adding the following functions to `lib.rs`:

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
    	id	            INTEGER NOT NULL,
    	name	        TEXT NOT NULL,
    	date_added	    REAL NOT NULL DEFAULT current_timestamp,
    	is_done	        NUMERIC NOT NULL DEFAULT 0,
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

The function blocks are pretty self explanatory, so I won't go into too much detail over them, but the `verify_db` function is interesting in that it shows the SQL table definitions for `todos` and the datatypes used for that particular DB flavour - it conveniently creates that table if it doesn't exist (i.e.: the first time this program is run).

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

A successfull call to `get_connection()` returns a database connection instance, that will be passed to the functions that perform database operations.

Commited code up to this point can be seen at [https://github.com/dezoito/rust-todo-list/tree/part2](https://github.com/dezoito/rust-todo-list/tree/part2).

---

<p>&nbsp;</p>

## Part 3: Adding Functionality

We now have to flesh out the functions that `add`, `remove`, `list`, `sort`, `toggle` the todo items or `reset` our database.

We can start to implement the methods for the `Todo` struct:

```rs
impl Todo {
    // Constructor for a new Todo instance
    pub fn new(id: i32, name: String, date_added: String, is_done: i32) -> Self {
        Todo {
            id,
            name,
            date_added,
            is_done,
        }
    }

    // Add a new todo to the database
    pub fn add(conn: &Connection, name: &str) -> Result<()> {
        conn.execute("INSERT INTO todo (name) VALUES (?)", &[name])?;
        Ok(())
    }

    // Other methods below
    // ...
}

```

The `new()` method accepts the data for each field and creates a new `Todo` instance.

The `add()` method performs a database operation, and, therefore expects a reference to a database `Connection object`. It also accepts the name for the `todo` (the remaining fields are handled automatically by the database).

We can now update `main()` with the proper call to `add()`:

```rs

fn main() -> Result<()> {
    // ...r().cloned().collect::<Vec<_>>().join(" ");

    match command.as_str() {
        "add" => {
            if suffix.as_str().is_empty() {
                help()?;
                std::process::exit(1);
            } else {
                Todo::add(&conn, suffix.as_str())?;
            }
            Ok(())
        }
        // ...
    }
}
```

To recap, if the user typed "`todo add go to class`" in the terminal, the match statement would route the execution to the `add()` method (the the `prefix` would be the strings left after the "add" command).

In this block we verify if the user effectively typed the name for a new `todo` (printing the "help" instructions otherwise), and call the `add()` implementation for a `Todo`, passing the connection reference and the `name`.

The other CRUD and auxiliary functions follow a similar pattern, so they won't be detailed here, but the complete source code for this part is available on this [Github Branch](https://github.com/dezoito/rust-todo-list/tree/part3).

Feel free to comment or ask questions if you don't understand how they work.

You can test the application by running the following commands:

```sh
cargo run add Task 1
cargo run add Task 2
cargo -q run list
```

... and you should get a result similar to:

```sh
TODO List (sorted by id):
   1 | Task 1                                       Pending  2023-11-01 15:39:54
   2 | Task 2                                       Pending  2023-11-01 15:39:54
```

---

<p>&nbsp;</p>

## Part 4: Testing the Application

The idiomatic way to test your Rust app is commonly done by writing your tests and assertions in the same file where the code that is being tested resides.

In this block, we will add tests for the CRUD functions defined in `lib.rs`:

We start by adding a `test` module at the end of the file:

```rs

#[cfg(test)]
mod tests {
    // "use" statements

    // Auxiliary functions

    // Tests and assertions
}

```

### Creating a Temporary Test Database

To test our CRUD and database related functions we could use the same SQLite file used in development, but that's ugly and less than ideal.

SQLite lets us create "in memory" databases, and we are going to use those when our test starts:

Our testing module will look like this:

```rs
#[cfg(test)]
mod tests {
    // "use" statements
    use super::*;
    use lazy_static::lazy_static;
    use std::sync::Mutex;

    // Auxiliary functions
    // Creates a persistant in memory db connection
    // creates tables if necessary
    lazy_static! {
        static ref DATABASE_CONNECTION: Mutex<Connection> = {
            let conn = Connection::open_in_memory().expect("Failed to create in-memory database");
            verify_db(&conn).expect("Cannot create tables");
            Mutex::new(conn)
        };
    }

    fn reset_db(conn: &Connection) -> Result<()> {
        conn.execute("DELETE FROM todo", ())?;
        Ok(())
    }

    // ....
}
```

In the `use super::*;` gives this module access to code defined in the parent scope.

The `lazy` and `mutex` imports are used to create a `lazy_static` database connection.

The `lazy_static!` macro block will allow the connection to be instantiated the first time the constant `DATABASE_CONNECTION` is accessed, and will persist in the global scope for the duration of the tests.

That block creates the connection to the in memory database and creates the tables if they don't exist, using `verify_db()`.

Finally, the database connection `conn`is wrapped in a `Mutex` before being returned, ensuring that only one thread can access the database connection at a time.

You can read more about `lazy_static` here: [Demystifying Rust’s lazy_static pattern](https://blog.logrocket.com/rust-lazy-static-pattern/)

We also added a `reset_db()` method, that "cleans" the database, so we can make sure that each test uses only data that was added in its own scope.

### Adding tests

Let's now add the first test to this module:

```rs
#[cfg(test)]
mod tests {
    // "use" statements
    use super::*;
    use lazy_static::lazy_static;
    use std::sync::Mutex;

    // Auxiliary functions

    // ...

    // Tests and assertions
    #[test]
    fn test_add_todo() {
        let conn = DATABASE_CONNECTION.lock().expect("Mutex lock failed");
        reset_db(&conn).expect("Messed up resetting the db");

        // Call the add function to add a todo
        let name = "Test Todo";
        Todo::add(&conn, name).expect("Failed to add todo");

        // Query the database to check if the todo was added
        let mut stmt = conn
            .prepare("SELECT COUNT(*) FROM todo WHERE name = ?")
            .expect("Failed to prepare statement");
        let count: i32 = stmt
            .query_row(&[name], |row| row.get(0))
            .expect("Failed to query database");

        assert_eq!(count, 1, "Todo was not added to the database");
    }


}
```

The `#[test]` attribute indicates to the Rust compiler that the associated function is a test function. When you run the test suite using a testing framework like `cargo test`, Rust will identify and execute functions marked with this attribute as part of the testing process.

In the following test, we are verifying whether our code can successfully add a todo.

Notice that we follow the Arrange, Act and Assert pattern of testing:

#### Arrange

We invoke our lazily initiated static database connection, then use it to reset the database to a clean state.

In some tests we can also add data and todo instances at this stage.

#### Act

This is where we perform the action that needs to be tested, such as adding a new `todo`:

```rs
    let name = "Test Todo";
    Todo::add(&conn, name).expect("Failed to add todo");
```

#### Assert

We perform a query to verify the addition of the todo to the database,

```rs
let mut stmt = conn
    .prepare("SELECT COUNT(*) FROM todo WHERE name = ?")
    .expect("Failed to prepare statement");
let count: i32 = stmt
    .query_row(&[name], |row| row.get(0))
    .expect("Failed to query database");
```

And then perform our assertion (there should be only a single row):

```rs
assert_eq!(count, 1, "Todo was not added to the database");
```

Displaying all tests here would ruin this article, but if you are curious they can be seen [here](https://github.com/dezoito/rust-todo-list/blob/c48060685748fb39a870ca84736f9665c3f42b6b/src/lib.rs#L212).

### Running tests

We can run our tests after manually using the `cargo test` command, but I prefer having a terminal open on my screen and having the tests run in `watch` mode as I develop, which reruns the tests whenever the code is saved:

```sh
cargo watch -c -x test
```

---

<p>&nbsp;</p>

## Part 5: Building the Executable

You can build the app by running

```sh
cargo build --release
```

This will create a release version of the `todo` executable in your projects `./target/release` folder.

In the repository, I added a convenient `build.sh` script that moves it to the PATH folder (in Linux). You can do something similar on your own for your particular OS.

After running the build script, I can run the application simply by typing the `todo` command, followed by the desired action, on the terminal:

```sh
❯ todo list
TODO List (sorted by id):
  24 | Task 1                                       Pending  2023-11-15 15:39:54
  25 | Task 2                                       Pending  2023-11-15 15:39:54

```

---

The final source code for this article is available in the project's [Github page](https://github.com/dezoito/rust-todo-list).

If you've made this far, please feel free to use the comments to ask questions or suggest improvements!
