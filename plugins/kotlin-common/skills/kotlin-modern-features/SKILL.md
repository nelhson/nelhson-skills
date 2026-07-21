---
name: kotlin-modern-features
description: Advisory catalog of modern Kotlin 2.x language features (context parameters, guard conditions, explicit backing fields, collection literals, etc.) with stability status and version requirements. Use when writing, reviewing, or refactoring Kotlin code in any project to suggest newer language features that simplify the code.
---

# Modern Kotlin Language Features (2.x era, current through 2.4.0)

Advisory "note" skill: when code being written or reviewed matches a **trigger** below, suggest (or apply, if refactoring was requested) the corresponding language feature — provided the project's Kotlin version supports it. This is language-level and framework-agnostic: it applies to Android, backend, and multiplatform code alike.

## Protocol

1. **Check the project's Kotlin version first** — look in `gradle/libs.versions.toml` (`kotlin = "..."`), `build.gradle.kts` (`kotlin("jvm") version`), or `gradle.properties`. Never suggest a feature the project's version cannot use.
2. **Stable features**: suggest or apply freely once the version allows.
3. **Experimental/preview features**: mention as an option and name the required compiler flag; do NOT add them to production code without the user's explicit go-ahead.
4. **Hint, not mandate**: during review, phrase as "Kotlin X.Y lets you simplify this with …". Only rewrite code when the user asked for a refactor.
5. Code examples for every feature live in [reference.md](reference.md) — read it only when actually applying a feature.

## Stable features

| Feature | Since | Trigger → suggestion |
|---|---|---|
| **Context parameters** | 2.4 | Same parameter (logger, transaction scope, DI'd service) threaded through many function signatures → declare `context(x: T)` and drop the explicit parameter |
| **Explicit backing fields** | 2.4 | `private val _x = MutableStateFlow(...)` + public `val x: StateFlow` pair → single property with `field =` |
| **`@all` meta-target** | 2.4 | Annotation needs to apply to param + property + field (e.g. validation annotations on data classes) → `@all:Annotation` |
| **New use-site annotation defaulting** | 2.4 | Annotations on constructor properties silently landing only on `param` → new defaults propagate to `param`, `property`, `field` |
| **Guard conditions in `when`** | 2.2 | Nested `if` inside a `when` branch, or duplicated branches differing by an extra condition → `branch if condition ->` |
| **Non-local `break`/`continue`** | 2.2 | Boolean flags or labeled returns to exit an outer loop from inside an inline lambda (`forEach`, `let`) → plain `break`/`continue` |
| **Multi-dollar string interpolation** | 2.2 | Strings full of escaped `$` (JSON schemas, regex, Gradle/GitHub templates) → `$$"..."` |
| **Nested type aliases** | 2.3 | A complex generic type repeated only within one class → `typealias` declared inside the class |
| **Data-flow exhaustiveness for `when`** | 2.3 | Redundant `else` branch after sealed-type/enum checks already proven exhaustive by earlier flow → drop the `else` |
| **`return` in expression bodies** | 2.3 | Early-return needs forced a block body on an otherwise single-expression function (explicit return type required) |

## Experimental / preview features (need opt-in — ask before using)

| Feature | Since | Flag / opt-in | Trigger |
|---|---|---|---|
| **Collection literals** | 2.4 | `-Xcollection-literals` | `listOf(...)` / `mutableListOf(...)` boilerplate → `["a", "b"]` |
| **Explicit context arguments** | 2.4 | `-Xexplicit-context-arguments` | Ambiguous overloads that differ only by context parameters → pass the context argument explicitly at the call site |
| **Unused return value checker** | 2.3 | `-Xreturn-value-checker=check` or `=full`; `@MustUseReturnValues` / `@IgnorableReturnValue` | Bugs from silently ignored return values (`Result`, immutable-collection ops) |
| **Context-sensitive resolution** | 2.2 (improved 2.3) | `-Xcontext-sensitive-resolution` (default-on parts in 2.3) | Repetitive enum/sealed-type prefixes in `when` branches → bare entry names |
| **Name-based destructuring** | 2.3.20 | experimental | Positional destructuring that breaks when data-class properties are reordered → match by property name |
| **Improved compile-time constants** | 2.4 | `-Xintrinsic-const-evaluation` | `const val` wants string functions, unsigned ops, or enum `.name` |

## On the roadmap

Rich errors (union-type error handling) and name-based destructuring stabilization are planned; Kotlin 2.4.20 is due Sep 2026 and 2.5.0 Dec 2026. If the project uses a Kotlin version newer than 2.4.0, check the matching "What's new" page at kotlinlang.org before assuming this catalog is complete.
