---
name: compose-performance-audit
description: Audit and improve Jetpack Compose runtime performance from code review and architecture. Use when asked to diagnose slow rendering, janky scrolling, excessive recompositions, or performance issues in Compose UI.
---

# Compose Performance Audit

## Overview

Audit Jetpack Compose view performance end-to-end, from instrumentation and baselining to root-cause analysis and concrete remediation steps.

## Workflow Decision Tree

- If the user provides code, start with "Code-First Review."
- If the user only describes symptoms, ask for minimal code/context, then do "Code-First Review."
- If code review is inconclusive, go to "Guide the User to Profile" and ask for Layout Inspector output or Perfetto traces.

## 1. Code-First Review

Collect:
- Target Composable code.
- Data flow: state, remember, derived state, ViewModel connections.
- Symptoms and reproduction steps.

Focus on:
- **Recomposition storms** from unstable parameters or broad state changes.
- **Unstable keys** in `LazyColumn`/`LazyRow` (`key` churn, missing keys).
- **Heavy work in composition** (formatting, sorting, filtering, object allocation).
- **Unnecessary recompositions** (missing `remember` for expensive values, unstable classes compared by instance equality).
- **Large images** without proper sizing or async loading.
- **Layout thrash** (deep nesting, intrinsic measurements, `SubcomposeLayout` misuse).

Provide:
- Likely root causes with code references.
- Suggested fixes and refactors.
- If needed, a minimal repro or instrumentation suggestion.

## 2. Guide the User to Profile

Explain how to collect data:
- Use **Layout Inspector** in Android Studio to see recomposition counts.
- Enable **Recomposition Highlights** in Compose tooling.
- Use **Perfetto** or **System Trace** for frame timing analysis.
- Check **Macrobenchmark** results for startup/scroll metrics.

Ask for:
- Layout Inspector screenshot showing recomposition counts.
- Perfetto trace or System Trace export.
- Device/OS/build configuration (debug vs release).

> **Important**: Ensure profiling is done on a **release build** with R8 enabled. Debug builds have significant overhead.

## 3. Analyze and Diagnose

Prioritize likely Compose culprits:
- **Recomposition storms** from unstable parameters or broad state changes.
- **Unstable keys** in lazy lists (`key` churn, index-based keys).
- **Heavy work in composition** (formatting, sorting, object allocation).
- **Missing `remember`** causing recreations on every recomposition.
- **Large images** without `Modifier.size()` constraints.
- **Unnecessary state reads** in wrong composition phases.

Summarize findings with evidence from traces/Layout Inspector.

## 4. Remediate

Apply targeted fixes:
- **Stabilize parameters**: Use `@Stable`/`@Immutable` on data classes (especially in non-Compose modules) so skipping uses `equals()` instead of instance equality.
- **Stabilize keys**: Use stable, unique IDs for `LazyColumn`/`LazyRow` items.
- **Defer state reads**: Use `derivedStateOf`, lambda-based modifiers, or `Modifier.drawBehind`.
- **Remember expensive computations**: Wrap in `remember { }` or `remember(key) { }`.
- **Skip recomposition**: Extract stable composables, use `key()` to control identity.
- **Async image loading**: Use Coil/Glide with proper sizing constraints.
- **Reduce layout complexity**: Flatten hierarchies, avoid deep nesting.

## Strong Skipping: What It Fixes (and What It Doesn't)

Strong skipping is **on by default** since the Compose compiler shipped with Kotlin 2.0.20. Calibrate advice to it before recommending fixes:

**Fixed automatically — do not recommend these anymore:**
- Wrapping every lambda in `remember { }`. Lambdas with unstable captures are now memoized by the compiler.
- `remember`-ing static `Modifier` chains "to avoid allocation". This was always noise; modifier chains are cheap and comparison is handled by the runtime.
- Making a composable skippable just because one parameter type is unstable — composables with unstable parameters are now skippable too.

**NOT fixed — still audit for these:**
- Unstable parameters are compared with **instance equality**, not `equals()`. If upstream code produces a *new* `List`, `copy()`-ed data class, or mapped collection on every emission, the composable still recomposes every time. Fix the producer (cache/`distinctUntilChanged`) or make the type stable so `equals()` is used.
- Classes from **modules without the Compose compiler** (pure Kotlin/Java modules, third-party libraries) and **interface/abstract types** are inferred unstable, so they fall into instance-equality comparison. `@Immutable`/`@Stable` annotations, stability configuration files, and `kotlinx.collections.immutable` types still matter: they upgrade comparison to `equals()`-based skipping.
- Lambdas capturing `var`s or unstable receivers can still defeat memoization — but verify with compiler reports before "fixing".

## Common Code Smells (and Fixes)

### Expensive work in composition

```kotlin
// BAD: Sorting on every recomposition
@Composable
fun ItemList(items: List<Item>) {
    val sorted = items.sortedBy { it.name } // Runs every recomposition
    LazyColumn { items(sorted) { ... } }
}

// GOOD: Use remember with key
@Composable
fun ItemList(items: List<Item>) {
    val sorted = remember(items) { items.sortedBy { it.name } }
    LazyColumn { items(sorted) { ... } }
}
```

### Missing keys in LazyColumn

```kotlin
// BAD: Index-based identity (causes recomposition on list changes)
LazyColumn {
    items(items) { item -> ItemRow(item) }
}

// GOOD: Stable key-based identity
LazyColumn {
    items(items, key = { it.id }) { item -> ItemRow(item) }
}
```

### Unstable data classes

```kotlin
// BAD: Unstable (contains List, which is not stable)
data class UiState(
    val items: List<Item>,
    val isLoading: Boolean
)

// GOOD: Mark as Immutable if truly immutable
@Immutable
data class UiState(
    val items: ImmutableList<Item>, // kotlinx.collections.immutable
    val isLoading: Boolean
)
```

### Reading state too early

```kotlin
// BAD: State read during composition (recomposes whole tree)
@Composable
fun AnimatedBox(scrollState: ScrollState) {
    val offset = scrollState.value // Recomposes on every scroll
    Box(modifier = Modifier.offset(y = offset.dp)) { ... }
}

// GOOD: Defer state read to layout/draw phase
@Composable
fun AnimatedBox(scrollState: ScrollState) {
    Box(modifier = Modifier.offset {
        IntOffset(0, scrollState.value) // Read in layout phase
    }) { ... }
}
```

### Unstable data crossing module boundaries

```kotlin
// BAD: UiState defined in a pure-Kotlin :domain module (no Compose compiler)
// is inferred unstable -> instance-equality comparison; a new .copy() or
// re-mapped list per emission recomposes consumers every time
data class UiState(val items: List<Item>)

// GOOD: Annotate in a Compose-aware module (or use a stability config file)
// so skipping falls back to equals()
@Immutable
data class UiState(val items: ImmutableList<Item>)
```

## Stability Checklist

| Type | Stable by Default? | Fix |
|------|-------------------|-----|
| Primitives (`Int`, `String`, `Boolean`) | Yes | N/A |
| `data class` with stable fields | Yes* | Ensure all fields are stable |
| `List`, `Map`, `Set` | **No** | Use `ImmutableList` from kotlinx (enables `equals()`-based skipping) |
| Classes with `var` properties | **No** | Use `@Stable` if externally stable |
| Classes from non-Compose modules / interfaces | **No** | `@Immutable`/`@Stable` or a stability configuration file |
| Lambdas | N/A | Memoized automatically under strong skipping; no manual `remember` needed |

## 5. Verify

Ask the user to:
- Re-run Layout Inspector and compare recomposition counts.
- Run Macrobenchmark and compare frame timing.
- Test on a real device with release build.

Summarize the delta (recomposition count, frame drops, jank) if provided.

## Outputs

Provide:
- A short metrics table (before/after if available).
- Top issues (ordered by impact).
- Proposed fixes with estimated effort.

## References

- [Jetpack Compose Performance](https://developer.android.com/develop/ui/compose/performance)
- [Compose Stability Explained](https://developer.android.com/develop/ui/compose/performance/stability)
- [Debugging Recomposition](https://developer.android.com/develop/ui/compose/tooling/layout-inspector)
- [Macrobenchmark](https://developer.android.com/topic/performance/benchmarking/macrobenchmark-overview)
