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

To sum it up:

- On startup, the app checks for a SQLite database in its data directory
- Creates it if missing, runs any pending migrations if needed
- Users get a working database without any configuration or intervention

> Note: The SQLite database will be stored in your system's application data directory:

> Windows: `C:\Users\<Username>\AppData\Roaming\com.github.dezoito.gridsearch`

> macOS: `~/Library/Application Support/com.github.dezoito.gridsearch`

> Linux: `~/.local/share/com.github.dezoito.gridsearch`

## Database Operations

Now that we have a database setup, we can implement the "Prompt Archive" feature. This allows users to store and manage reusable prompt templates.

> Our migration scripts include a set of pre-configured prompts that can be read by the application immediately.

In `src-tauri/src/commands/prompt.rs`, we first define a struct that represents our prompt data:

```rust
#[derive(Debug, FromRow, Serialize)]
pub struct Prompt {
    pub uuid: String,
    pub name: String,
    pub slug: String,
    pub prompt: String,
    pub date_created: i64,   // Unix timestamp
    pub last_modified: i64,  // Unix timestamp
}
```

This struct maps to our database schema, though for simplicity we've omitted some fields like `category` and `system_prompt`. The `FromRow` derive macro allows SQLx to automatically convert database rows into `Prompt` structs.

The function to retrieve prompts is implemented as a Tauri command:

```rust
#[tauri::command]
pub async fn get_all_prompts(state: tauri::State<'_, DatabaseState>) -> Result<Vec<Prompt>, Error> {
    let stmt = r#"
        SELECT
            uuid,
            name,
            slug,
            prompt,
            date_created,
            last_modified
        FROM prompts
        ORDER BY lower(name) ASC
    "#;
    let query = sqlx::query_as::<_, Prompt>(stmt);
    let pool = &state.0;
    let prompts = query.fetch_all(pool).await?;
    Ok(prompts)
}
```

The key line `let query = sqlx::query_as::<_, Prompt>(stmt);` tells SQLx to map each row from our SQL query to a `Prompt` struct. The `_` placeholder lets SQLx infer the database type (SQLite in our case) from the context.

The command is registered in `main.rs`:

```rust
app.invoke_handler(tauri::generate_handler![
    commands::get_all_prompts,
    // other commands...
])
```

Data from the above command is serialized as JSON and read by our React application (see [the source code](https://github.com/dezoito/ollama-grid-search/blob/main/src/components/queries/index.ts) for the complete implementation):

```typescript
import { invoke } from "@tauri-apps/api/tauri";
...
export async function get_all_prompts(): Promise<IPrompt[]> {
    const prompts = await invoke<IPrompt[]>("get_all_prompts");
    return prompts;
}
```

For the complete list of CRUD operations and working examples of parametrized queries, please refer to [the prompt.rs module](https://github.com/dezoito/ollama-grid-search/blob/main/src-tauri/src/commands/prompt.rs).

## Results & Insights

- Technical trade-offs
  - Compile-time verification considerations
  - Performance implications
- Implementation recommendations
- Future improvements
