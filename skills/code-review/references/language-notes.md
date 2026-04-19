# Language-Specific Review Notes

## Python

**Common pitfalls to check:**
- Mutable default arguments: `def foo(items=[])` — bug waiting to happen
- `except Exception` swallowing everything silently
- Using `is` for value comparison instead of `==`
- Missing `__slots__` in perf-sensitive classes
- `time.sleep()` in async code (use `asyncio.sleep`)
- Unintentional list/dict mutation (pass by reference)
- f-string in logging: `log.debug(f"...")` — evaluates even if log level is off

**Good patterns to praise:**
- Type hints with `from __future__ import annotations`
- Context managers (`with`) for resource cleanup
- `dataclasses` or `pydantic` for structured data
- `pathlib` over `os.path`

---

## JavaScript / TypeScript

**Common pitfalls:**
- `==` instead of `===`
- `async` function without `await` (returns Promise, not value)
- Unhandled promise rejections (missing `.catch()` or try/catch in async)
- `forEach` with async callbacks (doesn't await)
- Mutating state directly in React (should use setState / useState)
- Missing `?.` optional chaining on potentially null objects
- `var` instead of `const`/`let`

**TypeScript-specific:**
- `any` type usage — ask why and suggest proper typing
- Non-null assertion `!` without obvious guarantee
- Missing return type on exported functions
- `as unknown as X` double cast is a red flag

**Good patterns to praise:**
- Proper `Promise.all` for parallel async
- `readonly` arrays/objects where mutation isn't needed
- Discriminated unions for state machines
- Zod/io-ts for runtime validation

---

## Go

**Common pitfalls:**
- Ignoring returned errors: `result, _ := ...`
- Goroutine leak: goroutine started, no way to stop it
- `defer` inside a loop (runs at function exit, not loop iteration)
- Returning pointer to local variable (fine in Go, but verify intent)
- Using `interface{}` / `any` unnecessarily
- Missing `context` propagation in long-running operations

**Good patterns to praise:**
- Table-driven tests
- `errors.Is` / `errors.As` for error checking
- Proper `context.Context` threading
- `sync.WaitGroup` or `errgroup` for goroutine coordination

---

## SQL

**Critical checks:**
- String concatenation in queries → SQL injection
- `SELECT *` in production queries
- Missing `WHERE` on `UPDATE`/`DELETE`
- N+1: query inside a loop
- Missing indexes on JOIN/WHERE columns (if schema visible)
- Non-atomic operations that should be in a transaction

**Good patterns:**
- Parameterized queries / prepared statements
- Explicit column lists in `INSERT`
- Pagination with `LIMIT`/`OFFSET` or cursor-based

---

## Rust

**Common pitfalls:**
- Excessive `.clone()` — check if borrow would work
- `unwrap()` / `expect()` in non-test code
- Blocking inside async (`std::thread::sleep` vs `tokio::time::sleep`)
- Ignoring `#[must_use]` results

**Good patterns:**
- `?` operator for error propagation
- `From`/`Into` trait implementations
- Proper lifetime annotations

---

## Java / Kotlin

**Common pitfalls:**
- `NullPointerException` risks — unchecked null returns
- Resource leaks: streams/connections without try-with-resources
- `==` for String comparison (use `.equals()`)
- Raw types without generics
- Kotlin: `!!` non-null assertion without justification

---

## Shell / Bash

**Critical checks:**
- Unquoted variables: `rm $file` → `rm "$file"`
- Missing `set -e` / `set -u` / `set -o pipefail`
- `ls | xargs` instead of `find -exec` or `find -print0 | xargs -0`
- Command injection via unvalidated input
