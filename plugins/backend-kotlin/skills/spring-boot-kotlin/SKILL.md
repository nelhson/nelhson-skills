---
name: spring-boot-kotlin
description: Kotlin-idiomatic Spring Boot - constructor injection, compiler plugins (kotlin-spring/kotlin-jpa), data class usage rules, coroutine controllers, configuration properties, and null-safety at the web boundary. Use when writing or reviewing Spring Boot code in Kotlin.
---

# Spring Boot + Kotlin Expert Skill

This skill provides authoritative rules for writing idiomatic, production-quality Spring Boot applications in Kotlin — avoiding Java-style patterns that fight the language.

## Responsibilities

*   **Dependency Injection**: Constructor injection with immutable `val` properties.
*   **Compiler Plugins**: Correct use of `kotlin-spring` (all-open) and `kotlin-jpa` (no-arg).
*   **Data Modeling**: When to use data classes (DTOs: yes; JPA entities: no).
*   **Coroutines**: `suspend` controller functions and WebFlux/coroutine interop.
*   **Configuration**: Type-safe `@ConfigurationProperties` data classes.
*   **Validation & Null-Safety**: Enforcing non-null contracts at the web boundary.

## Applicability

Activate this skill when the user asks to:
*   "Create a Spring Boot service/controller/repository in Kotlin."
*   "Fix lateinit / autowired issues."
*   "Add JPA entities in Kotlin."
*   "Use coroutines with Spring."
*   "Bind configuration properties."

## Critical Rules & Constraints

### 1. Constructor Injection Only
*   **NEVER** use field injection (`@Autowired lateinit var`).
*   **ALWAYS** use constructor injection with `private val`. Single-constructor classes need no `@Autowired`.

```kotlin
// CORRECT
@Service
class UserService(
    private val userRepository: UserRepository,
    private val clock: Clock,
)

// INCORRECT
@Service
class UserService {
    @Autowired lateinit var userRepository: UserRepository
}
```

### 2. Compiler Plugins Instead of Manual `open`
*   Spring proxies require open classes; JPA requires no-arg constructors.
*   **ALWAYS** apply the `kotlin-spring` and `kotlin-jpa` Gradle plugins; **NEVER** hand-write `open` on beans or add fake default values to entities.

```kotlin
// build.gradle.kts
plugins {
    kotlin("plugin.spring") version kotlinVersion   // all-open for @Component, @Service, etc.
    kotlin("plugin.jpa") version kotlinVersion      // no-arg for @Entity
}
```

### 3. Data Classes: DTOs Yes, Entities No
*   **ALWAYS** use `@Serializable`/plain data classes for request/response DTOs.
*   **NEVER** use data classes for JPA entities — generated `equals`/`hashCode`/`toString` over mutable, lazily-loaded state breaks Hibernate semantics.

```kotlin
// CORRECT — entity as a plain class with id-based equality
@Entity
class UserEntity(
    @Id @GeneratedValue var id: Long? = null,
    var name: String,
    var email: String,
) {
    override fun equals(other: Any?) = other is UserEntity && id != null && id == other.id
    override fun hashCode() = javaClass.hashCode()
}

// CORRECT — DTO as a data class
data class UserResponse(val id: Long, val name: String, val email: String)
```

### 4. Coroutine Controllers
*   With WebFlux, **ALWAYS** prefer `suspend` functions and `Flow<T>` over `Mono`/`Flux` in Kotlin code.
*   With Spring MVC, `suspend` controller functions are supported (6.1+); keep blocking work off the event loop with `withContext(Dispatchers.IO)` only around genuinely blocking calls (e.g. JDBC).

```kotlin
@RestController
@RequestMapping("/users")
class UserController(private val service: UserService) {

    @GetMapping("/{id}")
    suspend fun get(@PathVariable id: Long): UserResponse = service.getById(id)

    @GetMapping
    fun stream(): Flow<UserResponse> = service.streamAll()
}
```

### 5. Type-Safe Configuration
*   **NEVER** scatter `@Value("\${...}")` strings across classes.
*   **ALWAYS** bind related settings into an immutable `@ConfigurationProperties` data class.

```kotlin
@ConfigurationProperties(prefix = "app.mail")
data class MailProperties(
    val host: String,
    val port: Int = 587,
    val from: String,
)
// + @EnableConfigurationProperties(MailProperties::class) or @ConfigurationPropertiesScan
```

### 6. Null-Safety at the Web Boundary
*   Declare DTO fields non-null wherever the contract requires them; combine with Bean Validation for messages.
*   **NEVER** use `!!` on request data — absent values must produce a 400, not a 500.

```kotlin
data class CreateUserRequest(
    @field:NotBlank val name: String,
    @field:Email val email: String,
)

@PostMapping
suspend fun create(@Valid @RequestBody request: CreateUserRequest): UserResponse =
    service.create(request)
```

### 7. Repositories & Transactions
*   Use Spring Data interfaces with Kotlin nullability: return `T?` instead of `Optional<T>`.
*   `@Transactional` only works on beans (proxied) — with coroutines use `@Transactional` on suspend functions only with reactive transaction managers; for JDBC keep transactional work inside a blocking service method wrapped by `withContext(Dispatchers.IO)`.

```kotlin
interface UserRepository : JpaRepository<UserEntity, Long> {
    fun findByEmail(email: String): UserEntity?   // not Optional<UserEntity>
}
```

### 8. Testing
*   **ALWAYS** use `@SpringBootTest` sparingly; prefer slice tests (`@WebMvcTest`/`@WebFluxTest`, `@DataJpaTest`) with MockK (`@MockkBean` via springmockk) over Mockito for Kotlin.
*   Use `runTest` for suspend service tests.
