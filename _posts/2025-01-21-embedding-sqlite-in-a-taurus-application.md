---
layout: post
comments: true
title: Embedding a SQLite database in a Taurus application
excerpt_separator: <!--more-->
---

In this text we discuss a straightforward method for adding data persistance to a Tauri application, using a SQLite database and the popular [SQLx crate](https://crates.io/crates/sqlx).

<!--more-->

## Introduction

- Quickly present the [Ollama Grid Search](https://github.com/dezoito/ollama-grid-search) project, and why it needed a database.

  - Store prompt templates
  - Store experiment results

- Explain why we chose SQLite and SQLx.

- Visually present the result: prompt archive

## Setup

- Explain the database.rs module, how it is integrated with main.rs, and how the SQLite database file is created.

- Explain how to setup migrations to create database structures like tables and indexes.

## Coding CRUD operations

- show the Tauri command that reads the list of prompts, and how it relates to the Typescript code

- show the code that creates a new prompt, both on the Tauri and React side of the application.

- point the user to other relevant functions in the repository.

## Compile-time verification

- Explain what this feature does and why we did not use it.

## Conclusion

- Reiterate the advantages of the technology choices
- Mention alternative crates like Diesel and SeaORM
