---
name: android-data-layer
description: Guidance on implementing the Data Layer using Repository pattern, Room (Local), and Retrofit (Remote) with offline-first synchronization. Use when building repositories, wiring Room + Retrofit together, adding offline-first caching/sync, or choosing local storage (Room vs DataStore).
---

# Android Data Layer & Offline-First

## Instructions

The Data Layer coordinates data from multiple sources.

### 1. Repository Pattern
*   **Role**: Single Source of Truth (SSOT).
*   **Logic**: The repository decides whether to return cached data or fetch fresh data.
*   **Implementation**:
    ```kotlin
    class NewsRepository @Inject constructor(
        private val newsDao: NewsDao,
        private val newsApi: NewsApi
    ) {
        // Expose data from Local DB as the source of truth, mapped to domain models
        val newsStream: Flow<List<News>> = newsDao.getAllNews()
            .map { entities -> entities.map(NewsEntity::toDomainModel) }

        // Sync operation: map network DTOs to Room entities before inserting.
        // Never insert DTOs directly — keep network and database models decoupled.
        suspend fun refreshNews() {
            val remoteNews: List<NetworkNews> = newsApi.fetchLatest()
            newsDao.insertAll(remoteNews.map(NetworkNews::toEntity))
        }
    }
    ```

### 2. Local Persistence (Room)
*   **Usage**: Primary cache and offline storage.
*   **Entities**: Define `@Entity` data classes.
*   **DAOs**: Return `Flow<T>` for observable data.
*   **Small key-value / typed settings**: Use **DataStore** instead of `SharedPreferences` (which it replaces): *Preferences DataStore* for simple key-value pairs, *Proto DataStore* for schema-backed typed objects. Both expose reads as `Flow` and use suspend functions for writes. Room remains the choice for relational/queryable data.

### 3. Remote Data (Retrofit)
*   **Usage**: Fetching data from backend.
*   **Response**: Use `suspend` functions in interfaces.
*   **Error Handling**: Wrap network calls in `try-catch` blocks or a `Result` wrapper to handle exceptions (NoInternet, 404, etc.) gracefully.

### 4. Synchronization
*   **Read**: "Stale-While-Revalidate". Show local data immediately, trigger a background refresh.
*   **Write**: "Outbox Pattern" (Advanced). Save local change immediately, mark as "unsynced", use `WorkManager` to push changes to server.

### 5. Dependency Injection
*   Bind Repository interfaces to implementations in a Hilt Module.
    ```kotlin
    @Binds
    abstract fun bindNewsRepository(impl: OfflineFirstNewsRepository): NewsRepository
    ```
