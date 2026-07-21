# Modern Kotlin features — before/after examples

Companion to [SKILL.md](SKILL.md). Each section shows the trigger pattern and the modernized form.

## Context parameters (stable, 2.4)

```kotlin
// Before: logger threaded through every signature
fun processOrder(order: Order, logger: Logger) {
    validate(order, logger)
    persist(order, logger)
}

// After
context(logger: Logger)
fun processOrder(order: Order) {
    logger.info("processing ${order.id}")
    validate(order)
    persist(order)
}

// Call site provides the context once:
with(logger) { processOrder(order) }
```

## Explicit backing fields (stable, 2.4)

```kotlin
// Before: _underscore pair
private val _city = MutableStateFlow("")
val city: StateFlow<String> get() = _city

// After: one property; smart-casts to the field type inside the class
val city: StateFlow<String>
    field = MutableStateFlow("")
```

## `@all` meta-target (stable, 2.4)

```kotlin
// Before: must repeat the annotation per use-site target
data class User(@param:Email @property:Email @field:Email val email: String)

// After
data class User(@all:Email val email: String)
```

## Guard conditions in `when` (stable, 2.2)

```kotlin
// Before: nested if inside a branch
when (status) {
    is Status.Error -> if (status.retryable) retry() else fail()
    else -> proceed()
}

// After
when (status) {
    is Status.Error if status.retryable -> retry()
    is Status.Error -> fail()
    else -> proceed()
}
```

## Non-local `break`/`continue` (stable, 2.2)

```kotlin
// Before: flag variable to escape the loop from a lambda
var found = false
for (page in pages) {
    page.items.forEach { if (it.matches(query)) { found = true; return@forEach } }
    if (found) break
}

// After: break directly from inside the inline lambda
for (page in pages) {
    page.items.forEach { if (it.matches(query)) break }
}
```

## Multi-dollar string interpolation (stable, 2.2)

```kotlin
// Before: escaped dollars everywhere
val schema = "{ \"\$id\": \"https://example.com\", \"price\": \"\${'$'}9.99\" }"

// After: single $ is literal, $$ interpolates
val schema = $$"""{ "$id": "https://example.com", "value": $$amount }"""
```

## Nested type aliases (stable, 2.3)

```kotlin
class Dijkstra {
    typealias VisitedNodes = Set<Node>   // scoped to this class only
    private fun step(visited: VisitedNodes) { /* ... */ }
}
```

## Data-flow exhaustiveness for `when` (stable, 2.3)

```kotlin
sealed interface Result { object Ok : Result; object Err : Result }

fun handle(r: Result) {
    if (r is Result.Err) return
    when (r) {
        is Result.Ok -> println("ok")
        // no `else` needed — compiler knows Err is impossible here
    }
}
```

## K2 smart cast improvements (stable, 2.0)

```kotlin
// Before: manual cast after a check combined with ||
if (obj is Cat || obj is Dog) {
    (obj as Animal).feed()
}

// After: smart cast to the common supertype
if (obj is Cat || obj is Dog) {
    obj.feed()
}
```

## `@SubclassOptInRequired` (stable, 2.1)

```kotlin
@RequiresOptIn(level = RequiresOptIn.Level.WARNING)
annotation class UnstableApi

@SubclassOptInRequired(UnstableApi::class)
interface CoreLibraryApi   // implementing it requires @OptIn(UnstableApi::class)
```

## `Enum.entries` (stable, 2.0)

```kotlin
// Before: allocates a new array on every call
RGB.values().forEach { println(it) }

// After: cached immutable list
RGB.entries.forEach { println(it) }

// Generic version: enumValues<T>() → enumEntries<T>()
```

## Common `AutoCloseable` + `use` (stable, 2.0)

```kotlin
// Works in multiplatform common code, not just JVM
val resource = AutoCloseable { writer.flushAndClose() }
resource.use { /* ... */ }   // closed even on exception
```

## `Path` tree traversal (stable, 2.1)

```kotlin
// Before: recursive java.io.File walking by hand
// After:
Path("dir").walk().filter { it.extension == "log" }.forEach(::println)
```

## `Base64` API (stable, 2.2)

```kotlin
// Before: java.util.Base64 (JVM-only)
// After: multiplatform
Base64.Default.encode(bytes)      // "Zm8="
Base64.UrlSafe.encode(bytes)      // URL/filename-safe variant
Base64.decode("Zm8=")
```

## `HexFormat` API (stable, 2.2)

```kotlin
// Before: String.format("%02x", b) loops
// After:
93.toHexString()                  // "0000005d"
"5d".hexToByte()
byteArrayOf(0xDE.toByte()).toHexString()
```

## `kotlin.time.Instant` / `Clock` (stable, 2.3)

```kotlin
// Before: untestable, JVM-only
val now = System.currentTimeMillis()

// After: multiplatform and injectable for tests
class OrderService(private val clock: Clock = Clock.System) {
    fun stamp(): Instant = clock.now()
}
// test: pass a fake Clock returning a fixed Instant
```

## `kotlin.uuid.Uuid` (stable, 2.4)

```kotlin
// Before: java.util.UUID (JVM-only)
// After: multiplatform
val id = Uuid.random()                    // random v4 uuid (stable)
val parsed = Uuid.parseOrNull(input)      // null instead of exception
val ordered = uuidA < uuidB               // Comparable since 2.4

// NOT yet: Uuid.generateV4() / Uuid.generateV7() (time-ordered, DB-index
// friendly) are still @ExperimentalUuidApi — do not suggest them until stable.
```

---

# Watchlist examples — DO NOT USE

Everything below is experimental/preview. Kept only for reference so the examples are ready when a feature is promoted to stable; never suggest or apply these in a project.

## Collection literals (experimental, 2.4, `-Xcollection-literals`)

```kotlin
val fruit = ["apple", "banana", "cherry"]          // List<String>
val shapes: MutableList<String> = ["triangle"]     // target-typed
```

## Unused return value checker (experimental, 2.3, `-Xreturn-value-checker=check|full`)

```kotlin
@MustUseReturnValues
class Greeter { fun greet(name: String): String = "Hello, $name" }

greeter.greet("A")        // warning: result ignored
val _ = greeter.greet("A") // explicit discard, no warning

@IgnorableReturnValue      // opt a function out
fun <T> MutableList<T>.addQuietly(e: T): Boolean = add(e)
```

## Context-sensitive resolution (experimental, 2.2+, `-Xcontext-sensitive-resolution`)

```kotlin
enum class Problem { CONNECTION, AUTHENTICATION, DATABASE }

fun message(problem: Problem) = when (problem) {
    CONNECTION -> "connection"      // no Problem. prefix needed
    AUTHENTICATION -> "authentication"
    DATABASE -> "database"
}
```

## Name-based destructuring (experimental, 2.3.20)

```kotlin
data class Person(val name: String, val age: Int)

// Before: positional — silently wrong if properties are reordered
val (age, name) = person   // bug: age gets name's value

// After: matched by property name, order-independent
val (name = n, age = a) = person
```

## Improved compile-time constants (experimental, 2.4, `-Xintrinsic-const-evaluation`)

```kotlin
const val UPPER = "hello".uppercase()   // string functions
const val NAME = MyEnum.VARIANT.name    // enum .name
const val SUM = 5u + 3u                 // unsigned ops
```
