---
title: "Updating a Rust Project to Latest Dependencies"
date: 2025-11-05
draft: false
tags: ["rust", "cursor", "dependencies"]
categories: ["rust", "ai"]
---

I recently updated an older Rust project ([termdex](https://github.com/mtclinton/termdex), a terminal-based Pokédex) to use the latest package versions. Here's what I ran into and how I fixed it.

## The Starting Point

The project was using older versions of most dependencies:
- `ratatui` 0.26 (still named `tui` in Cargo.toml)
- `diesel` 2.0.0
- `crossterm` 0.26
- `reqwest` 0.11
- And several others

The goal was to get everything up to the latest stable versions while keeping the project functional.

## Major Updates

### Ratatui Migration

The biggest change was moving from `ratatui` 0.27 to 0.29. The library had several API changes:

- **Frame generics**: `Frame<B>` became just `Frame` (no generic parameter)
- **Area method**: `f.size()` → `f.area()`
- **Cursor positioning**: `f.set_cursor()` → `f.set_cursor_position()` (takes a tuple now)
- **Text API**: `Spans` was replaced with `Line`. Code like `Text::from(Spans::from(...))` became `Text::from(vec![Line::from(...)])`
- **Wrap API**: `Wrap { trim: true }` works, but `Wrap::trim()` doesn't exist in 0.29

### Diesel Macros

Diesel 2.1 deprecated the old `#[table_name = "..."]` syntax. I updated all models to use the new format:

```rust
// Old
#[table_name = "pokemon"]
pub struct NewPokemon { ... }

// New
#[diesel(table_name = pokemon)]
pub struct NewPokemon { ... }
```

### Version Conflicts

The trickiest part was resolving version conflicts between dependencies:

- `tui-input` 0.8 used `crossterm` 0.27, but `ratatui` 0.29 needs `crossterm` 0.28
- Solution: Updated `tui-input` to 0.14, which supports `crossterm` 0.28
- `ansi-to-tui` had similar issues—upgraded from 3.0 to 7.0 to match `ratatui` 0.29

### Docker Updates

The Dockerfiles needed Rust 1.86 to compile the latest `diesel_cli` (2.3.3). Updated both Dockerfiles from `rust:1.82` to `rust:1.86`.

## Common Issues

1. **Borrow checker**: Had to clone `Text` objects before passing them to widgets when they're used later
2. **Unused variables**: Some loops had `index` that wasn't used, but in one case it was actually needed for array indexing
3. **Deprecated methods**: Various method renames and API changes that required grep searches through the codebase

## The Process

I used Cursor's AI assistant to help with this. The workflow was:
1. Update Cargo.toml with target versions
2. Run `cargo build` to see errors
3. Fix compilation errors one by one
4. Repeat until it builds

The AI was helpful for finding all the places that needed changes (like searching for `f.size()` or `#[table_name`), but you still need to understand the codebase structure to make the right fixes.

## Final Result

![termdex](termdex.gif)

After updating:
- `ratatui`: 0.27 → 0.29
- `crossterm`: 0.27 → 0.28
- `diesel`: 2.0.0 → 2.1
- `reqwest`: 0.11 → 0.12
- `tui-input`: 0.8 → 0.14
- `ansi-to-tui`: 3.0 → 7.0

Everything compiles and works. The warnings about unused imports and results are still there, but those are non-blocking.

## Takeaways

1. **Read the error messages carefully**—they often point to exactly what needs changing
2. **Update in stages**—don't try to update everything at once
3. **Version conflicts are common**—dependencies often have their own version requirements
4. **Keep Dockerfiles in sync**—Rust version requirements can break builds

The whole process took a couple of hours, mostly spent fixing API changes and resolving version conflicts. The code is now on modern, maintained versions of all dependencies.

