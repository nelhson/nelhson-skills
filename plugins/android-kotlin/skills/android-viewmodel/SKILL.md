---
name: android-viewmodel
description: Best practices for implementing Android ViewModels, focused on StateFlow for UI state and modeling one-off events as state. Use when creating or refactoring a ViewModel, exposing UI state to Compose/Views, or handling one-off UI events (snackbars, toasts, navigation signals).
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

### 2. One-Off Events (Model as State)
*   **What**: Transient signals like "Show Snackbar", "Show Toast", "Navigate to Screen".
*   **Default**: Model events **as state** that the UI consumes and then asks the ViewModel to clear. Fire-and-forget event flows can silently **drop events** when no collector is active (configuration change, backgrounded UI, process death), which is why current Google guidance treats them as an anti-pattern.
    ```kotlin
    data class UiState(
        val userMessage: String? = null, // one-off event, held in state
        // ... other state
    )

    fun onLoginFailed() {
        _uiState.update { it.copy(userMessage = "Login failed") }
    }

    // Called by the UI after the message has been shown
    fun userMessageShown() {
        _uiState.update { it.copy(userMessage = null) }
    }
    ```
    The UI observes `userMessage`, shows the snackbar/toast (or navigates), then calls `userMessageShown()` to clear it. The event survives rotation and process recreation, and cannot be lost.
*   **Alternative (use with caution)**: `SharedFlow<UiEvent>` with `replay = 0`.
    ```kotlin
    private val _uiEvent = MutableSharedFlow<UiEvent>(replay = 0)
    val uiEvent: SharedFlow<UiEvent> = _uiEvent.asSharedFlow()
    ```
    *   Send with `.emit(event)` (suspend) or `.tryEmit(event)`.
    *   **Caveat**: emissions with no active collector are dropped. Only acceptable when losing the event is harmless (e.g., a purely cosmetic animation trigger). Never use it for navigation or anything that *must* happen.

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
