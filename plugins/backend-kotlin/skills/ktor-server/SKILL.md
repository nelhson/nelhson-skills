---
name: ktor-server
description: Best practices for building Ktor server applications in Kotlin - routing DSL organization, plugin installation, dependency injection with Koin, configuration, and testing with testApplication. Use when writing or reviewing Ktor services.
---

# Ktor Server Expert Skill

This skill provides authoritative rules and patterns for building production-quality Ktor server applications in Kotlin.

## Responsibilities

*   **Routing**: Organizing the routing DSL into feature-based `Route` extension functions.
*   **Plugins**: Installing and configuring core plugins (`ContentNegotiation`, `StatusPages`, `CallLogging`, `Authentication`).
*   **Serialization**: Type-safe request/response bodies with `kotlinx.serialization`.
*   **Dependency Injection**: Wiring services and repositories with Koin.
*   **Configuration**: Externalizing settings via `application.yaml` / environment variables.
*   **Testing**: Writing integration tests with `testApplication`.

## Applicability

Activate this skill when the user asks to:
*   "Create a Ktor server / REST API / endpoint."
*   "Add authentication / JWT to a Ktor app."
*   "Structure or refactor Ktor routes."
*   "Handle errors / status codes in Ktor."
*   "Test a Ktor application."

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
fun Application.module() {
    configurePlugins()
    routing {
        userRoutes(get())   // Koin: get() resolves UserService
        orderRoutes(get())
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

### 5. Dependency Injection with Koin
*   **NEVER** instantiate services/repositories inline in routes.
*   **ALWAYS** declare them in Koin modules and resolve via `get()` / constructor injection.

```kotlin
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
    port: $PORT:8080
jwt:
  secret: $JWT_SECRET
  issuer: "nelhson"
```

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
