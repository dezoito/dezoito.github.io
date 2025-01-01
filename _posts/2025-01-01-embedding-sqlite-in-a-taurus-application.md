---
layout: post
comments: true
title: Embedding a SQLite database in a Taurus Application
excerpt_separator: <!--more-->
---

In this text we discuss implementing SQLite storage in a Tauri application using the [SQLx crate](https://crates.io/crates/sqlx), demonstrating practical approaches and solutions for local data persistence.

<!--more-->

## The Problem

[Ollama Grid Search](https://github.com/dezoito/ollama-grid-search) helps users evaluate LLM models by running experiments that compare different prompts and inference parameters. Built with Tauri and Rust, it provides an interface for testing multiple configurations and analyzing their results.

Initially, the experiment results were stored in JSON files and there was no way for the user to store the prompts they used the most.

Clearly, the application needed a proper database to handle prompt templates and experiment results with better organization, concurrent access, and data integrity. The next section explains our choice of SQLite and SQLx as the technical solution.

[<img src="https://raw.githubusercontent.com/dezoito/ollama-grid-search/refs/heads/main/screenshots/prompt-archive.png" alt="Settings" width="720">](https://github.com/dezoito/ollama-grid-search?tab=readme-ov-file#prompt-archive)

## Solution Design

SQLite is a natural fit for Tauri applications: it's serverless, requires zero configuration, and the database is contained in a single file. This matches Tauri's goal of creating lightweight, distributable desktop applications.

SQLx was chosen as our Rust SQL toolkit because it provides:

- Native SQL support without requiring a custom query syntax.
- Async support out of the box.
- Database migrations can be implemented easily.
- A clean API for parameterized queries.

While alternatives like Diesel and SeaORM offer similar features, SQLx's straightforward approach made it ideal for our needs.

Let's look at how we implemented these tools in practice.

## Implementation

The database integration starts with adding SQLx to our project's dependencies in `Cargo.toml`, specifying SQLite support and required features:

```toml
sqlx = { version = "0.8.1", features = ["runtime-tokio", "sqlite", "chrono"] }
```

In `db.rs`, we define our database architecture: a `Database` struct manages the SQLite connection pool, with its location determined by the application's data directory. The implementation ensures the database file is created if it doesn't exist and uses Write-Ahead Logging for better concurrent access:

```rust
// From db.rs
let connection_options = sqlx::sqlite::SqliteConnectOptions::new()
    .filename(&db_path)
    .create_if_missing(true)
    .journal_mode(sqlx::sqlite::SqliteJournalMode::Wal);
```

The database is initialized in `main.rs` before setting up the rest of the application:

```rust
// From main.rs
fn main() {

  ...

  let app = builder.setup(|app| {
      let handle = app.handle();
      // Initialize database
      tauri::async_runtime::block_on(async move {
          let database = db::Database::new(&handle)
              .await
              .expect("failed to initialize database");
          // Store database pool in app state
          app.manage(db::DatabaseState(database.pool));
      });
      Ok(())
  });

  ...
```

For schema management, SQLx's migration system provides version control of our database structure. Migrations are SQL files stored in the `./migrations` directory, named with a timestamp prefix. These are automatically run during database initialization in `db.rs`:

```rust
sqlx::migrate!("./migrations").run(&pool).await?;
```

Here's our first migration file (`./migrations/20240101_initial_schema.sql`), which creates the tables for storing prompts:

```sql
-- Create prompts table
CREATE TABLE IF NOT EXISTS prompts (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    title TEXT NOT NULL,
    content TEXT NOT NULL,
    system_prompt TEXT,
    category TEXT,
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
    updated_at DATETIME DEFAULT CURRENT_TIMESTAMP
);

-- Create index on title for faster searches
CREATE INDEX idx_prompts_title ON prompts(title);
```

The complete set of migrations can be found in the [project's repository](https://github.com/dezoito/ollama-grid-search/tree/main/src-tauri/migrations).

To sum it up, when the application starts, it checks for a SQLite database file in the application's data directory. If the file doesn't exist, it creates one automatically. Then, regardless of whether the database is new or existing, it runs any pending migrations to ensure the schema is up to date. This setup provides a zero-configuration experience for users while maintaining a robust database structure. The whole process happens transparently during application initialization, requiring no user intervention.

====

- CRUD operations implementation
- Practical example: prompt archive feature
  - Backend code (Rust/SQLx)
  - Frontend integration (TypeScript/React)
  - Error handling patterns

## Results & Insights

- Technical trade-offs
  - Compile-time verification considerations
  - Performance implications
- Implementation recommendations
- Future improvements

```

```
