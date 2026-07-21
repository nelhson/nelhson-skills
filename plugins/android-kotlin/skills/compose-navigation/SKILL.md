---
name: compose-navigation
description: Implement navigation with Navigation Compose 2.x (NavHost/NavController). Use when asked to set up NavHost-based navigation, pass arguments between screens, handle deep links, or structure multi-screen apps with the androidx.navigation 2.x library. Not for Jetpack Navigation 3 (NavDisplay/backstack-as-state).
---

# Compose Navigation

## Overview

Implement type-safe navigation in Jetpack Compose applications using the Navigation Compose library. This skill covers NavHost setup, argument passing, deep links, nested graphs, adaptive navigation, and testing.

> **Scope — Navigation 2 vs Navigation 3**: This skill covers **Navigation Compose 2.x** (`NavHost`/`NavController`, `androidx.navigation:navigation-compose`). For **Jetpack Navigation 3** (`NavDisplay`, backstack-as-state, Scenes) — including migrating to it — use the external `android-official` **navigation-3** skill instead.

## Setup

Add the Navigation Compose dependency:

```kotlin
// build.gradle.kts
dependencies {
    implementation("androidx.navigation:navigation-compose:2.9.7")
    
    // For type-safe navigation (recommended)
    implementation("org.jetbrains.kotlinx:kotlinx-serialization-json:1.11.0")
}

// Enable serialization plugin (match your project's Kotlin version)
plugins {
    kotlin("plugin.serialization") version "2.4.0"
}
```

---

## Core Concepts

### 1. Define Routes (Type-Safe)

Use `@Serializable` data classes/objects for type-safe routes:

```kotlin
import kotlinx.serialization.Serializable

// Simple screen (no arguments)
@Serializable
object Home

// Screen with required argument
@Serializable
data class Profile(val userId: String)

// Screen with optional argument
@Serializable
data class Settings(val section: String? = null)

// Screen with multiple arguments
@Serializable
data class ProductDetail(val productId: String, val showReviews: Boolean = false)
```

### 2. Create NavController

```kotlin
@Composable
fun MyApp() {
    val navController = rememberNavController()
    
    AppNavHost(navController = navController)
}
```

### 3. Create NavHost

```kotlin
@Composable
fun AppNavHost(
    navController: NavHostController,
    modifier: Modifier = Modifier
) {
    NavHost(
        navController = navController,
        startDestination = Home,
        modifier = modifier
    ) {
        composable<Home> {
            HomeScreen(
                onNavigateToProfile = { userId ->
                    navController.navigate(Profile(userId))
                }
            )
        }
        
        composable<Profile> { backStackEntry ->
            val profile: Profile = backStackEntry.toRoute()
            ProfileScreen(userId = profile.userId)
        }
        
        composable<Settings> { backStackEntry ->
            val settings: Settings = backStackEntry.toRoute()
            SettingsScreen(section = settings.section)
        }
    }
}
```

---

## Detailed Patterns

Read `reference.md` in this skill directory when applying any of the following:

- **Navigation patterns** — basic navigation, navigate with options (`popUpTo`, `launchSingleTop`, `restoreState`), bottom navigation
- **Argument handling** — `toRoute()` in composables, `SavedStateHandle.toRoute<T>()` in ViewModels, passing IDs vs objects
- **Deep links** — `navDeepLink<T>()`, manifest configuration, PendingIntents for notifications
- **Nested navigation** — nested graphs for flows (e.g., auth)
- **Adaptive navigation** — `NavigationSuiteScaffold`
- **Testing** — `TestNavHostController` setup and assertions

---

## Critical Rules

### DO

- Use `@Serializable` routes for type safety
- Pass only IDs/primitives as arguments
- Use `popUpTo` with `launchSingleTop` for bottom navigation
- Extract `NavHost` to a separate composable for testability
- Use `SavedStateHandle.toRoute<T>()` in ViewModels

### DON'T

- Pass complex objects as navigation arguments
- Create `NavController` inside `NavHost`
- Navigate in `LaunchedEffect` without proper keys
- Forget `FLAG_IMMUTABLE` for PendingIntents (Android 12+)
- Use string-based routes (legacy pattern)

---

## References

- [Navigation with Compose](https://developer.android.com/develop/ui/compose/navigation)
- [Type-Safe Navigation](https://developer.android.com/guide/navigation/design#compose)
- [Pass Data Between Destinations](https://developer.android.com/guide/navigation/navigation-pass-data)
- [Test Navigation](https://developer.android.com/guide/navigation/navigation-testing)
