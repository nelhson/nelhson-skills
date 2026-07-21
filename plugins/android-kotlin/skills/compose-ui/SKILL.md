---
name: compose-ui
description: Best practices for authoring Jetpack Compose components - state hoisting, modifier conventions, theming, and previews. Use when writing new Composable functions, hoisting state, structuring modifier parameters, applying Material theming, or adding previews. Not for performance audits, navigation, or image loading (separate skills cover those).
---

# Jetpack Compose Best Practices

## Instructions

Follow these guidelines to create performant, reusable, and testable Composables.

### 1. State Hoisting (Unidirectional Data Flow)
Make Composables **stateless** whenever possible by moving state to the caller.

*   **Pattern**: Function signature should usually look like:
    ```kotlin
    @Composable
    fun MyComponent(
        value: String,              // State flows down
        onValueChange: (String) -> Unit, // Events flow up
        modifier: Modifier = Modifier // Standard modifier parameter
    )
    ```
*   **Benefit**: Decouples the UI from simple state storage, making it easier to preview and test.
*   **ViewModel Integration**: The screen-level Composable retrieves state from the ViewModel (`viewModel.uiState.collectAsStateWithLifecycle()`) and passes it down.

### 2. Modifiers
*   **Default Parameter**: Always provide a `modifier: Modifier = Modifier` as the first optional parameter.
*   **Application**: Apply this `modifier` to the *root* layout element of your Composable.
*   **Ordering matters**: `padding().clickable()` is different from `clickable().padding()`. Generally apply layout-affecting modifiers (like padding) *after* click listeners if you want the padding to be clickable.

### 3. Performance
*   For recomposition, stability, and rendering-performance concerns, use the **compose-performance-audit** skill — it is the single source of truth for Compose performance guidance.

### 4. Theming and Resources
*   Use `MaterialTheme.colorScheme` and `MaterialTheme.typography` instead of hardcoded colors or text styles.
*   Organize simple UI components into specific files (e.g., `DesignSystem.kt` or `Components.kt`) if they are shared across features.
*   **Material 3 Expressive**: expressive theming and component variants (`MaterialExpressiveTheme`, flexible app bars, etc.) ship in the `material3` 1.5.0-alpha line (graduating from `@ExperimentalMaterial3ExpressiveApi` since 1.5.0-alpha23); on the stable 1.4.x line, stick to standard M3 APIs.

### 5. Previews
*   Create a private preview function for every public Composable.
*   Use `@Preview(showBackground = true)` and include Light/Dark mode previews if applicable.
*   Pass dummy data (static) to the stateless Composable for the preview.
