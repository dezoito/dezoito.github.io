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

- Database module architecture
- Schema design and migrations
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
