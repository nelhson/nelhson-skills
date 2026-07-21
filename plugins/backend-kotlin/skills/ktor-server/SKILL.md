---
name: ktor-server
description: Best practices for building Ktor server applications in Kotlin - routing DSL organization, plugin installation, dependency injection (built-in DI plugin or Koin), configuration, and testing with testApplication. Use when writing or reviewing Ktor services.
---

# Ktor Server Expert Skill

This skill provides authoritative rules and patterns for building production-quality Ktor server applications in Kotlin.

## Responsibilities

*   **Routing**: Organizing the routing DSL into feature-based `Route` extension functions.
*   **Plugins**: Installing and configuring core plugins (`ContentNegotiation`, `StatusPages`, `CallLogging`, `Authentication`).
*   **Serialization**: Type-safe request/response bodies with `kotlinx.serialization`.
*   **Dependency Injection**: Wiring services and repositories with the built-in DI plugin (Ktor 3.2+) or Koin.
*   **Configuration**: Externalizing settings via `application.yaml` / environment variables.
*   **Testing**: Writing integration tests with `testApplication`.

## Critical Rules & Constraints

### 1. Feature-Based Route Organization
*   **NEVER** put all routes in a single `routing { }` block in `Application.module()`.
*   **ALWAYS** define one `Route` extension function per feature, in its own file, and compose them in the module.

```kotlin
// CORRECT — features/user/UserRoutes.kt
fun Route.userRoutes(service: UserService) {
    route("/users") {
        get { call.respond(service.getAll()) }
        get("/{id}") {
            val id = call.parameters["id"]?.toLongOrNull()
                ?: throw BadRequestException("Invalid id")
            call.respond(service.getById(id))
        }
        post {
            val request = call.receive<CreateUserRequest>()
            call.respond(HttpStatusCode.Created, service.create(request))
        }
    }
}

// Application.kt
suspend fun Application.module() {
    configurePlugins()
    val userService: UserService = dependencies.resolve()   // built-in DI — see rule 5
    val orderService: OrderService = dependencies.resolve()
    routing {
        userRoutes(userService)
        orderRoutes(orderService)
    }
}
```

### 2. Content Negotiation & DTOs
*   **ALWAYS** install `ContentNegotiation` with `kotlinx.serialization` JSON.
*   **ALWAYS** use `@Serializable` data classes as request/response DTOs. Never expose database entities directly.

```kotlin
install(ContentNegotiation) {
    json(Json {
        ignoreUnknownKeys = true
        explicitNulls = false
    })
}

@Serializable
data class CreateUserRequest(val name: String, val email: String)
```

### 3. Centralized Error Handling with StatusPages
*   **NEVER** wrap every route handler in `try-catch`.
*   **ALWAYS** install `StatusPages` and map domain exceptions to typed error responses.

```kotlin
@Serializable
data class ErrorResponse(val code: String, val message: String)

install(StatusPages) {
    exception<NotFoundException> { call, cause ->
        call.respond(HttpStatusCode.NotFound, ErrorResponse("NOT_FOUND", cause.message ?: ""))
    }
    exception<BadRequestException> { call, cause ->
        call.respond(HttpStatusCode.BadRequest, ErrorResponse("BAD_REQUEST", cause.message ?: ""))
    }
    exception<Throwable> { call, cause ->
        call.application.log.error("Unhandled exception", cause)
        call.respond(HttpStatusCode.InternalServerError, ErrorResponse("INTERNAL", "Internal server error"))
    }
}
```

*   **ALWAYS** validate request bodies with the `RequestValidation` plugin instead of ad-hoc checks inside handlers; map its `RequestValidationException` to a 400 in `StatusPages`.

```kotlin
install(RequestValidation) {
    validate<CreateUserRequest> { req ->
        if (req.email.isBlank()) ValidationResult.Invalid("email must not be blank")
        else ValidationResult.Valid
    }
}

// in StatusPages:
exception<RequestValidationException> { call, cause ->
    call.respond(HttpStatusCode.BadRequest, ErrorResponse("VALIDATION", cause.reasons.joinToString()))
}
```

*   For type-safe routing (compile-checked paths and parameters), consider the `Resources` plugin: `@Resource("/users/{id}") class UserById(val id: Long)` + `get<UserById> { }`.

### 4. Authentication (JWT)
*   Configure JWT in one place; protect routes with `authenticate("auth-jwt") { }` blocks.
*   **NEVER** hardcode secrets — read them from configuration/environment.

```kotlin
install(Authentication) {
    jwt("auth-jwt") {
        realm = config.property("jwt.realm").getString()
        verifier(
            JWT.require(Algorithm.HMAC256(secret))
                .withIssuer(issuer)
                .build()
        )
        validate { credential ->
            if (credential.payload.getClaim("userId").asLong() != null)
                JWTPrincipal(credential.payload) else null
        }
    }
}
```

### 5. Dependency Injection
*   **NEVER** instantiate services/repositories inline in routes.
*   **For new services on Ktor 3.2+**, prefer the built-in DI plugin (`io.ktor:ktor-server-di`, package `io.ktor.server.plugins.di`): register with `dependencies { provide { } }`, resolve with `dependencies.resolve()`.
*   Since 3.2, application modules can be `suspend` functions — `resolve()` suspends until the dependency graph is ready, so async initialization (e.g. a datasource) needs no workarounds.
*   **Koin remains acceptable**, especially in existing codebases already using it or when you need its wider ecosystem — declare beans in Koin modules and resolve via `get()`.

```kotlin
// Built-in DI (Ktor 3.2+)
fun Application.diModule() {
    dependencies {
        provide<UserRepository> { ExposedUserRepository(resolve()) }
        provide { UserService(resolve()) }
    }
}

// Modules may be suspend (3.2+); resolve() suspends until dependencies are ready
suspend fun Application.userModule() {
    val service: UserService = dependencies.resolve()
    routing { userRoutes(service) }
}

// Koin alternative (existing codebases)
val appModule = module {
    single<UserRepository> { ExposedUserRepository(get()) }
    single { UserService(get()) }
}

fun Application.configureKoin() {
    install(Koin) { modules(appModule) }
}
```

### 6. Configuration
*   **ALWAYS** externalize configuration in `application.yaml` (or HOCON) with environment-variable overrides; never hardcode ports, URLs, or credentials.

```yaml
ktor:
  deployment:
    port: "$PORT:8080"   # YAML supports env var with inline default ($VAR:default)
jwt:
  secret: "$JWT_SECRET"
  issuer: "nelhson"
```

*   Note: the inline `:default` fallback is a YAML-config feature. In HOCON (`application.conf`) there is no inline default — set the default separately and override with `port = ${?PORT}`.

### 7. Structured Concurrency in Handlers
*   Route handlers are already suspend functions — call suspend services directly; **NEVER** launch coroutines in `GlobalScope` from a handler.
*   For fire-and-forget work that must outlive the request, use an injected application-level `CoroutineScope`.

### 8. Testing with testApplication
*   **ALWAYS** test endpoints with `testApplication` and a configured test client; assert both status codes and response bodies.

```kotlin
@Test
fun `GET users returns 200 with list`() = testApplication {
    application { module() }
    val client = createClient {
        install(io.ktor.client.plugins.contentnegotiation.ContentNegotiation) { json() }
    }
    val response = client.get("/users")
    assertEquals(HttpStatusCode.OK, response.status)
    assertTrue(response.body<List<UserResponse>>().isNotEmpty())
}
```
