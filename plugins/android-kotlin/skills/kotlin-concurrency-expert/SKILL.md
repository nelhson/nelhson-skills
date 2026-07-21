---
name: kotlin-concurrency-expert
description: Kotlin Coroutines review and remediation for Android. Use when asked to review concurrency usage, fix coroutine-related bugs, improve thread safety, or resolve lifecycle issues in Kotlin/Android code.
---

# Kotlin Concurrency Expert

## Overview

Review and fix Kotlin Coroutines issues in Android codebases by applying structured concurrency, lifecycle safety, proper scoping, and modern best practices with minimal behavior changes.

The canonical coroutine rules and code patterns (dispatcher injection, `repeatOnLifecycle`, state encapsulation, cancellation handling, `callbackFlow`, testing) live in the `android-coroutines` skill — consult it for the full rule set; this skill covers the review and remediation workflow.

## Workflow

### 1. Triage the Issue

- Capture the exact error, crash, or symptom (ANR, memory leak, race condition, incorrect state).
- Check project coroutines setup: `kotlinx-coroutines-android` version, `lifecycle-runtime-ktx` version.
- Identify the current scope context (`viewModelScope`, `lifecycleScope`, custom scope, or none).
- Confirm whether the code is UI-bound (`Dispatchers.Main`) or intended to run off the main thread (`Dispatchers.IO`, `Dispatchers.Default`).
- Verify Dispatcher injection patterns for testability.

### 2. Apply the Smallest Safe Fix

Prefer edits that preserve existing behavior while satisfying structured concurrency and lifecycle safety.

Common fixes:

- **ANR / Main thread blocking**: Move heavy work to `withContext(Dispatchers.IO)` or `Dispatchers.Default`; ensure suspend functions are main-safe.
- **Memory leaks / zombie coroutines**: Replace `GlobalScope` with a lifecycle-bound scope (`viewModelScope`, `lifecycleScope`, or injected `applicationScope`).
- **Lifecycle collection issues**: Replace deprecated `launchWhenStarted` with `repeatOnLifecycle(Lifecycle.State.STARTED)`.
- **State exposure**: Encapsulate `MutableStateFlow` / `MutableSharedFlow`; expose read-only `StateFlow` or `Flow`.
- **CancellationException swallowing**: Ensure generic `catch (e: Exception)` blocks rethrow `CancellationException`.
- **Non-cooperative cancellation**: Add `ensureActive()` or `yield()` in tight loops for cooperative cancellation.
- **Callback APIs**: Convert listeners to `callbackFlow` with proper `awaitClose` cleanup.
- **Hardcoded Dispatchers**: Inject `CoroutineDispatcher` via constructor for testability.

For the good/bad code examples backing each of these fixes, see the corresponding rules in the `android-coroutines` skill.

### 3. Verify the Fix

- Confirm the symptom is gone (no ANR, leak, or stale state) without changing observable behavior.
- Ensure fixed code is covered by a test using `runTest` and an injected `TestDispatcher` (pattern in the `android-coroutines` skill).

## Scope Guidelines (Diagnosis Aid)

| Scope | Use When | Lifecycle |
|-------|----------|-----------|
| `viewModelScope` | ViewModel operations | Cleared with ViewModel |
| `lifecycleScope` | UI operations in Activity/Fragment | Destroyed with lifecycle owner |
| `repeatOnLifecycle` | Flow collection in UI | Started/Stopped with lifecycle state |
| `applicationScope` (injected) | App-wide background work | Application lifetime |
| `GlobalScope` | **NEVER USE** | Breaks structured concurrency |

## Reference Material

- [Kotlin Coroutines Best Practices](https://developer.android.com/kotlin/coroutines/coroutines-best-practices)
- [StateFlow and SharedFlow](https://developer.android.com/kotlin/flow/stateflow-and-sharedflow)
- [repeatOnLifecycle API](https://developer.android.com/topic/libraries/architecture/coroutines#repeatOnLifecycle)
