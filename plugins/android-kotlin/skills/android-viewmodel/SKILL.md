---
name: android-viewmodel
description: Best practices for implementing Android ViewModels, specifically focused on StateFlow for UI state and SharedFlow for one-off events.
---

# Android ViewModel & State Management

## Instructions

Use `ViewModel` to hold state and business logic. It must outlive configuration changes.

### 1. UI State (StateFlow)
*   **What**: Represents the persistent state of the UI (e.g., `Loading`, `Success(data)`, `Error`).
*   **Type**: `StateFlow<UiState>`.
*   **Initialization**: Must have an initial value.
*   **Exposure**: Expose as a read-only `StateFlow` backing a private `MutableStateFlow`.
    ```kotlin
    private val _uiState = MutableStateFlow<UiState>(UiState.Loading)
    val uiState: StateFlow<UiState> = _uiState.asStateFlow()
    ```
*   **Updates**: Update state using `.update { oldState -> ... }` for thread safety.

### 2. One-Off Events (SharedFlow)
*   **What**: Transient events like "Show Toast", "Navigate to Screen", "Show Snackbar".
*   **Type**: `SharedFlow<UiEvent>`.
*   **Configuration**: Must use `replay = 0` to prevent events from re-triggering on screen rotation.
    ```kotlin
    private val _uiEvent = MutableSharedFlow<UiEvent>(replay = 0)
    val uiEvent: SharedFlow<UiEvent> = _uiEvent.asSharedFlow()
    ```
*   **Sending**: Use `.emit(event)` (suspend) or `.tryEmit(event)`.

### 3. Collecting in UI
*   **Compose**: Use `collectAsStateWithLifecycle()` for `StateFlow`.
    ```kotlin
    val state by viewModel.uiState.collectAsStateWithLifecycle()
    ```
    For `SharedFlow`, use `LaunchedEffect` with `LocalLifecycleOwner`.
*   **Views (XML)**: Use `repeatOnLifecycle(Lifecycle.State.STARTED)` within a coroutine.

### 4. Scope
*   Use `viewModelScope` for all coroutines started by the ViewModel.
*   Ideally, specific operations should be delegated to UseCases or Repositories.

### 5. No Android Framework Dependencies
*   **Never** reference `android.content.Context` (or `Activity`, `View`, `Resources`, `Uri`, `SharedPreferences`, etc.) or any other Android-specific class/library inside a ViewModel. It couples business logic to the framework, breaks separation of concerns, and makes the ViewModel impossible to unit test on the JVM.
*   **Instead**:
    *   Wrap platform needs (string resources, file access, preferences) behind an abstraction (interface) in the domain/data layer and inject it via Hilt.
    *   Resolve resources/strings in the UI layer (Composable/Fragment) and pass primitive results into the ViewModel, or expose string-resource IDs from the state and resolve them in the UI.
    *   Use UseCases/Repositories for anything that touches the platform.
*   The only sanctioned exception is `AndroidViewModel(application)` when an `Application` context is unavoidable — prefer the abstraction approach instead.
