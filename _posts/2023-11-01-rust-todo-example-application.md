---
layout: post
comments: true
title: Rust Todo SQL Example Application
excerpt_separator: <!--more-->
---

Coming across [Top 15 Rust Projects To Elevate Your Skills](https://zerotomastery.io/blog/rust-practice-projects/), I decided to build a Todo CLI application (something something unbeaten path) and defined a few goals/requirements after checking some of their suggested crates and examples:

- Use a database to persist todos (I used SQLite).
- Minimize the use of `unwrap()` and `expect()` calls.
- Include relevant working test cases, even if they are not 100% necessary.

![](https://github.com/dezoito/dezoito.github.io/blob/master/public/images/todo-demo?raw=true)

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

Turns out that was a great choice for the scope of a first project, as all your data can be stored in a simple file and you can even use an "in memory" database when running tests with very little configuration.

### Code structure.

Keeping it simple, we have two Rust files in the `src/` folder:

`main.rs`: Entry point that accepts arguments and routes execution to functions accordingly.

`lib.rs`: Contains the functions needed to connect and interact with the database and print the records. Also contains the tests.
