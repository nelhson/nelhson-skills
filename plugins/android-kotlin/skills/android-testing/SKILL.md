---
name: android-testing
description: Android test strategy and patterns — Unit, Hilt integration, Compose UI, Turbine Flow assertions, and Roborazzi screenshot tests. Use when deciding what/how to test, writing tests for ViewModels, DAOs, Flows, or Compose UI, or reviewing test coverage. (For installing test libraries and wiring test infrastructure, defer to the android-official:testing-setup skill.)
---

# Android Testing Strategies

This skill provides expert guidance on testing modern Android applications, inspired by "Now in Android". It covers **Unit Tests**, **Hilt Integration Tests**, **Compose UI Tests**, **Flow assertions with Turbine**, and **Screenshot Testing**.

> Scope: this skill owns test *strategy and patterns*. For installing test libraries and setting up test infrastructure/harnesses, use the external `android-official:testing-setup` skill.

## Testing Pyramid

1.  **Unit Tests**: Fast, isolate logic (ViewModels, Repositories).
2.  **Integration Tests**: Test interactions (Room DAOs, Retrofit vs MockWebServer).
3.  **UI/Screenshot Tests**: Verify UI correctness (Compose).

## Dependencies (`libs.versions.toml`)

Ensure you have the right testing dependencies.

```toml
[libraries]
junit4 = { module = "junit:junit", version = "4.13.2" }
kotlinx-coroutines-test = { group = "org.jetbrains.kotlinx", name = "kotlinx-coroutines-test", version.ref = "kotlinxCoroutines" }
androidx-test-ext-junit = { group = "androidx.test.ext", name = "junit", version = "1.3.0" }
espresso-core = { group = "androidx.test.espresso", name = "espresso-core", version = "3.7.0" }
compose-ui-test = { group = "androidx.compose.ui", name = "ui-test-junit4" }
turbine = { group = "app.cash.turbine", name = "turbine", version.ref = "turbine" }
hilt-android-testing = { group = "com.google.dagger", name = "hilt-android-testing", version.ref = "hilt" }
roborazzi = { group = "io.github.takahirom.roborazzi", name = "roborazzi", version.ref = "roborazzi" }
```

## Compose UI Testing

Use `createComposeRule()` to test composables in isolation (or `createAndroidComposeRule<ComponentActivity>()` when an Activity/theme resources are needed).

```kotlin
class NewsScreenTest {

    @get:Rule
    val composeTestRule = createComposeRule()

    @Test
    fun clickingHeadline_showsDetail() {
        composeTestRule.setContent {
            MyTheme { NewsScreen(state = fakeState) }
        }

        // Prefer semantics-based finders (what the user/TalkBack sees)...
        composeTestRule.onNodeWithText("Headline").performClick()
        // ...fall back to testTag for nodes without stable text
        composeTestRule.onNodeWithTag("news_list").assertIsDisplayed()

        // Wait for async UI (e.g., loading indicator to disappear)
        composeTestRule.waitUntil(timeoutMillis = 2_000) {
            composeTestRule.onAllNodesWithTag("loading").fetchSemanticsNodes().isEmpty()
        }
    }
}
```

*   Finders: `onNodeWithText`, `onNodeWithContentDescription`, `onNodeWithTag`; assertions: `assertIsDisplayed()`, `assertIsEnabled()`; actions: `performClick()`, `performTextInput()`.
*   Set tags via `Modifier.testTag("news_list")`; use `waitUntil { ... }` instead of `Thread.sleep`.

## Flow Assertions with Turbine

Use **Turbine** (`app.cash.turbine`) to assert on `Flow`/`StateFlow` emissions inside `runTest` instead of hand-rolling collectors.

```kotlin
@Test
fun `uiState emits Loading then Success`() = runTest {
    viewModel.uiState.test {
        assertEquals(UiState.Loading, awaitItem())

        viewModel.loadData()

        assertEquals(UiState.Success(expectedData), awaitItem())
        cancelAndIgnoreRemainingEvents()
    }
}
```

*   `awaitItem()` suspends until the next emission; `awaitComplete()` / `awaitError()` assert terminal events.
*   Turbine fails the test on unconsumed events — end with `cancelAndIgnoreRemainingEvents()` for hot flows (`StateFlow`/`SharedFlow`) that never complete.

## Screenshot Testing with Roborazzi

Screenshot tests ensure your UI doesn't regress visually. NiA uses **Roborazzi** because it runs on the JVM (fast) without needing an emulator.

### Setup

1.  Add the plugin to `libs.versions.toml`:
    ```toml
    [plugins]
    roborazzi = { id = "io.github.takahirom.roborazzi", version.ref = "roborazzi" }
    ```
2.  Apply it in your module's `build.gradle.kts`:
    ```kotlin
    plugins {
        alias(libs.plugins.roborazzi)
    }
    ```

### Writing a Screenshot Test

```kotlin
@RunWith(AndroidJUnit4::class)
@GraphicsMode(GraphicsMode.Mode.NATIVE)
@Config(sdk = [36], qualifiers = RobolectricDeviceQualifiers.Pixel5) // Robolectric 4.16 supports up to SDK 36 (requires Java 21)
class MyScreenScreenshotTest {

    @get:Rule
    val composeTestRule = createAndroidComposeRule<ComponentActivity>()

    @Test
    fun captureMyScreen() {
        composeTestRule.setContent {
            MyTheme {
                MyScreen()
            }
        }

        composeTestRule.onRoot()
            .captureRoboImage()
    }
}
```

## Hilt Testing

Use `HiltAndroidRule` to inject dependencies in tests.

```kotlin
@HiltAndroidTest
class MyDaoTest {

    @get:Rule
    var hiltRule = HiltAndroidRule(this)

    @Inject
    lateinit var database: MyDatabase
    private lateinit var dao: MyDao

    @Before
    fun init() {
        hiltRule.inject()
        dao = database.myDao()
    }
    
    // ... tests
}
```

## Running Tests

*   **Unit**: `./gradlew test`
*   **Screenshots**: `./gradlew recordRoborazziDebug` (to record) / `./gradlew verifyRoborazziDebug` (to verify)
