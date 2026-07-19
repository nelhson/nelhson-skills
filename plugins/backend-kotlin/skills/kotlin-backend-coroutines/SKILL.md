---
name: kotlin-backend-coroutines
description: Server-side Kotlin coroutine patterns shared by Ktor and Spring - dispatcher selection, structured concurrency for request handling, parallel fan-out, Flow streaming, timeouts, retries, and cancellation. Use for concurrency questions in Kotlin backends.
---

# Kotlin Backend Coroutines Expert Skill

This skill provides authoritative rules for using Kotlin Coroutines on the server (Ktor, Spring WebFlux/MVC, plain JVM services). It complements framework-specific skills with the shared concurrency layer.

## Responsibilities

*   **Dispatcher Selection**: Choosing `Dispatchers.IO` vs `Default` vs custom pools.
*   **Blocking Interop**: Safely wrapping JDBC and other blocking APIs.
*   **Fan-Out**: Parallel calls with `async`/`awaitAll` and `supervisorScope`.
*   **Streaming**: `Flow` for SSE / chunked responses.
*   **Resilience**: Timeouts, retries with backoff, cancellation on client disconnect.
*   **Context Propagation**: MDC / tracing context across coroutines.

## Applicability

Activate this skill when the user asks to:
*   "Call several services in parallel."
*   "Wrap a blocking database/HTTP call."
*   "Stream results to the client."
*   "Add a timeout / retry."
*   "Fix a coroutine leak or hung request in a backend."

## Critical Rules & Constraints

### 1. Dispatcher Selection
*   **`Dispatchers.IO`** — blocking I/O only (JDBC, file access, blocking clients).
*   **`Dispatchers.Default`** — CPU-bound work (serialization of large payloads, crypto, compression).
*   **NEVER** run blocking calls on the server engine's event-loop threads (Netty/CIO) — it starves the whole server.
*   Inject dispatchers via constructor for testability (same rule as on Android).

### 2. Wrap Blocking Calls with withContext
*   **ALWAYS** confine blocking APIs (JDBC, JPA, legacy SDKs) inside `withContext(Dispatchers.IO)` at the repository layer, so all callers get a main-safe suspend function.

```kotlin
class JdbcUserRepository(
    private val db: DataSource,
    private val ioDispatcher: CoroutineDispatcher = Dispatchers.IO,
) : UserRepository {
    override suspend fun findById(id: Long): User? = withContext(ioDispatcher) {
        db.connection.use { /* blocking JDBC */ }
    }
}
```

### 3. Parallel Fan-Out
*   **ALWAYS** use `coroutineScope { async { } }` + `awaitAll()` for independent calls whose results are all required — a failure cancels the siblings and propagates.
*   Use `supervisorScope` only when partial failure is acceptable, and handle each failure explicitly.

```kotlin
// All-or-nothing (default)
suspend fun loadDashboard(userId: Long): Dashboard = coroutineScope {
    val profile = async { profileService.get(userId) }
    val orders = async { orderService.recent(userId) }
    Dashboard(profile.await(), orders.await())
}

// Partial results tolerated
suspend fun loadWidgets(ids: List<Long>): List<Widget?> = supervisorScope {
    ids.map { id ->
        async {
            runCatching { widgetService.get(id) }
                .onFailure { log.warn("widget $id failed", it) }
                .getOrNull()
        }
    }.awaitAll()
}
```

### 4. Flow for Streaming Responses
*   **ALWAYS** expose unbounded/large result sets as `Flow<T>` and stream them (SSE, chunked JSON) instead of materializing full lists.
*   Apply `flowOn(ioDispatcher)` at the producer; keep operators (map/filter) dispatcher-agnostic.

```kotlin
fun events(): Flow<Event> =
    repository.streamEvents()
        .map { it.toDto() }
        .flowOn(Dispatchers.IO)
```

### 5. Timeouts and Retries
*   **ALWAYS** bound outbound calls with `withTimeout`; convert `TimeoutCancellationException` to a domain error at the boundary.
*   Retry only idempotent operations, with exponential backoff and a retry cap.

```kotlin
suspend fun <T> retrying(
    times: Int = 3,
    initialDelay: Duration = 100.milliseconds,
    block: suspend () -> T,
): T {
    var delayTime = initialDelay
    repeat(times - 1) {
        try { return block() } catch (e: IOException) {
            delay(delayTime)
            delayTime *= 2
        }
    }
    return block()   // last attempt: exception propagates
}

val user = withTimeout(2.seconds) { retrying { client.fetchUser(id) } }
```

### 6. Cancellation on Client Disconnect
*   Request-handler scopes are cancelled when the client disconnects — long operations must stay **cooperative**: call suspend functions, or check `ensureActive()` inside loops.
*   **NEVER** swallow `CancellationException` — always rethrow it from generic `catch (e: Exception)` blocks.

```kotlin
try {
    process()
} catch (e: CancellationException) {
    throw e            // let cancellation propagate
} catch (e: Exception) {
    log.error("processing failed", e)
    throw ProcessingException(e)
}
```

### 7. Scope Discipline
*   **NEVER** use `GlobalScope` or create ad-hoc `CoroutineScope(Dispatchers.IO)` inside request handlers.
*   Work that must outlive the request (audit logging, cache warm-up) goes to an injected application-level scope with a `SupervisorJob` and an exception handler, created and closed with the application lifecycle.

```kotlin
class ApplicationScope : AutoCloseable {
    private val job = SupervisorJob()
    val scope = CoroutineScope(job + Dispatchers.Default +
        CoroutineExceptionHandler { _, e -> log.error("background task failed", e) })
    override fun close() = job.cancel()
}
```

### 8. Context Propagation (MDC / Tracing)
*   Coroutines hop threads, so thread-local MDC is lost by default.
*   **ALWAYS** use `MDCContext()` (kotlinx-coroutines-slf4j) — or your tracing library's coroutine support — when launching coroutines that must keep request ids in logs.

```kotlin
withContext(MDCContext()) {
    service.handle(request)   // log lines keep requestId from MDC
}
```

### 9. Testing
*   Use `runTest` with injected `StandardTestDispatcher`/`UnconfinedTestDispatcher` instead of real dispatchers.
*   Assert cancellation behavior explicitly (e.g. `job.cancelAndJoin()` then verify cleanup).
