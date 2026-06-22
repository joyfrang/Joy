# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Status

This is a **language design proposal** — not an implemented compiler or runtime. The repository contains the Joy language and web framework specification. There are no build, lint, test, or run commands. The Demo/ directory is illustrative only (no functional compiler exists yet).

## Core Documents

- **`README.md`** — The canonical language/framework specification. All syntax, semantics, and design decisions live here. This should be treated as the source of truth for how Joy works. When asked about Joy features, consult README.md first.
- **`OG.md`** — Archived original proposal from early 2024. **Never read or modify this file.** It exists for historical preservation only. The README.md supersedes it entirely.

## Repository Structure

- `README.md` — Complete language + framework spec (comments, data modeling, validation, closures, contracts, generics, error handling, configuration system, queries, memory model, concurrency, caching, imports, web framework, CLI tools)
- `OG.md` — Archived original proposal (do not touch)
- `Demo/WebApplication/` — Illustrative Joy web app showing entrypoint, routing, layouts, views, islands, database config, and assets
- `Demo/Bundle/` — Illustrative third-party bundle showing how reusable packages could be structured
- `.gitignore` — Ignores `*.py` files

## Demo Conventions (for writing illustrative Joy code)

- `Wapp entry()` is the app entrypoint, configured in `entry.joy`
- Pages are `View` components under `wapp/`, annotated with `#page("/route")`
- File-based routing: `wapp/blog/[permalink]/[permalink].joy` maps to `/blog/:permalink`
- `Layout` components accept `Renderable children` and wrap page content
- `Island` components contain interactive client-side logic
- `database/config.joy` configures the database provider via `configureDatabase()`
- Joy file extension is `.joy`
- Imports use `bring joy:...`, `bring author:...`, or `bring local:...`

## Language Design Principles (for making design decisions)

- **"Joyful Programming"** — Pragmatic over dogmatic; borrow from FP, OOP, or any paradigm that makes sense for the use case
- **No magic** — Validation is plain function calls, not annotations. Errors are values (`bomb<T, E>`), not exceptions. Control flow is explicit
- **Compiler as ally** — The compiler owns the language, ORM, and framework together; it can statically validate queries, cache tags, and provide deep tooling
- **Secure by default** — Functions are server-side unless explicitly marked as client-side (`Island`)
- **Value semantics** — Clone-by-default with structural sharing (COW); no ownership cycles allowed
- **Structured concurrency** — `branch` and `server` blocks are tied to parent scopes
- **Extensible protocol** — `setup(namespace/path for Thing)` is the unified configuration mechanism for database, JSON, caching, and third-party bundles

## TODO Convention

TODOs live at the bottom of `README.md` with `[x]` for completed and `[ ]` for pending. This is the project roadmap — add new items here rather than in separate issue trackers or inline comments.