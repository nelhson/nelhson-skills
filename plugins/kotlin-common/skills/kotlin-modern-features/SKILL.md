---
name: kotlin-modern-features
description: Advisory catalog of modern Kotlin 2.x language features (context parameters, guard conditions, explicit backing fields, smart casts) and stdlib APIs (Enum.entries, Base64, Uuid, kotlin.time) with stability status and version requirements. Use when writing, reviewing, or refactoring Kotlin code in any project to suggest newer language features that simplify the code.
---

# Modern Kotlin Language Features (2.x era, current through 2.4.0)

Advisory "note" skill: when code being written or reviewed matches a **trigger** below, suggest (or apply, if refactoring was requested) the corresponding language feature — provided the project's Kotlin version supports it. This is language-level and framework-agnostic: it applies to Android, backend, and multiplatform code alike.

## Protocol

1. **Check the project's Kotlin version first** — look in `gradle/libs.versions.toml` (`kotlin = "..."`), `build.gradle.kts` (`kotlin("jvm") version`), or `gradle.properties`. Never suggest a feature the project's version cannot use.
2. **Stable features**: suggest or apply freely once the version allows.
3. **Experimental/preview features**: **do NOT use or suggest them.** The watchlist below exists only to track them until Kotlin marks them stable, at which point they get promoted into the stable table.
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
| **Data-flow exhaustiveness for `when`** | 2.1/2.3 | Redundant `else` branch after sealed-type/enum checks already proven exhaustive (2.1: sealed with generics; 2.3: data-flow analysis) → drop the `else` |
| **`return` in expression bodies** | 2.3 | Early-return needs forced a block body on an otherwise single-expression function (explicit return type required) |
| **K2 smart cast improvements** | 2.0 | Explicit casts or `!!` after type checks — including checks combined with `\|\|`, checks on local `val`s, and in `catch`/`finally` → drop the cast, rely on smart cast |
| **`@SubclassOptInRequired`** | 2.1 | Library exposes an unstable interface/class that users shouldn't extend yet → require explicit opt-in to subclass |
| **Extra compiler checks `-Wextra`** | 2.1 | Project wants stricter static analysis → enable `-Wextra` (15 opt-in code-quality warnings) |

## Stable standard library

| Feature | Since | Trigger → suggestion |
|---|---|---|
| **`Enum.entries` / `enumEntries<T>()`** | 2.0 | `MyEnum.values()` or `enumValues<T>()` → `entries` (returns a cached list, no array allocation per call) |
| **Common `AutoCloseable` + `use`** | 2.0 | Manual try/finally resource cleanup, especially in multiplatform common code → `AutoCloseable.use { }` |
| **`Path` tree traversal** | 2.1 | Manual recursive directory walking with `java.io.File` → `java.nio.file.Path` extensions `walk()`, `fileVisitor()`, `visitFileTree()` |
| **`Base64` API** | 2.2 | `java.util.Base64` or hand-rolled encoding → multiplatform `kotlin.io.encoding.Base64` (`Default`/`UrlSafe`/`Mime`/`Pem`) |
| **`HexFormat` API** | 2.2 | Manual hex conversion (`String.format("%02x")`, lookup tables) → `toHexString()` / `hexToByteArray()` |
| **`kotlin.time.Instant` / `Clock`** | 2.3 | `System.currentTimeMillis()` or `java.time.Instant` in shared/common code; untestable time access → inject `Clock`, use `kotlin.time.Instant` |
| **`kotlin.uuid.Uuid`** | 2.4 | `java.util.UUID` in common/multiplatform code → `kotlin.uuid.Uuid` (`Uuid.random()` for v4, `parse`/`parseOrNull`, `<`/`>` comparison). Note: `generateV4()`/`generateV7()` are still experimental (`@ExperimentalUuidApi`) — do not suggest them |

## Watchlist: experimental / preview features — DO NOT USE

Not to be suggested or applied in any project. Tracked here only so they can be moved to the stable table above once a Kotlin release stabilizes them.

| Feature | Since | Flag / opt-in | Trigger |
|---|---|---|---|
| **Collection literals** | 2.4 | `-Xcollection-literals` | `listOf(...)` / `mutableListOf(...)` boilerplate → `["a", "b"]` |
| **Explicit context arguments** | 2.4 | `-Xexplicit-context-arguments` | Ambiguous overloads that differ only by context parameters → pass the context argument explicitly at the call site |
| **Unused return value checker** | 2.3 | `-Xreturn-value-checker=check` or `=full`; `@MustUseReturnValues` / `@IgnorableReturnValue` | Bugs from silently ignored return values (`Result`, immutable-collection ops) |
| **Context-sensitive resolution** | 2.2 (improved 2.3) | `-Xcontext-sensitive-resolution` (still experimental as of 2.4) | Repetitive enum/sealed-type prefixes in `when` branches → bare entry names |
| **Name-based destructuring** | 2.3.20 | experimental | Positional destructuring that breaks when data-class properties are reordered → match by property name |
| **Improved compile-time constants** | 2.4 | `-Xintrinsic-const-evaluation` | `const val` wants string functions, unsigned ops, or enum `.name` |

## Keeping this catalog up to date

Rich errors (union-type error handling) and name-based destructuring stabilization are planned — verify against the current roadmap at kotlinlang.org/docs/roadmap.html. By the documented cadence (tooling releases ~3 months after a language release, language releases every ~6 months; 2.4.0 shipped Jun 2026 and 2.4.20-Beta1 is out), 2.4.20 is expected around Sep 2026 and 2.5.0 around Dec 2026 — treat these as estimates, not announced dates. When asked to update this skill (or when a project uses a Kotlin version newer than 2.4.0), check the matching "What's new" page at kotlinlang.org: move watchlist entries that became stable into the stable table, and add newly announced features to the watchlist.
