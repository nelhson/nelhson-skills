---
name: coil-compose
description: Expert guidance on using Coil for image loading in Jetpack Compose. Use this when asked about loading images from URLs, handling image states, or optimizing image performance in Compose.
---

# Coil for Jetpack Compose

## Instructions

When implementing image loading in Jetpack Compose, use **Coil 3** (Coroutines Image Loader). It is the recommended library for Compose due to its efficiency, seamless integration, and multiplatform support.

### 1. Setup (Coil 3)

Coil 3 uses new Maven coordinates (`io.coil-kt.coil3`) and packages (`coil3.*`), and networking is a separate artifact — without one, URL loading silently does nothing:

```kotlin
implementation("io.coil-kt.coil3:coil-compose:3.5.0")
// Pick ONE network backend:
implementation("io.coil-kt.coil3:coil-network-okhttp:3.5.0") // Android/OkHttp projects
// implementation("io.coil-kt.coil3:coil-network-ktor3:3.5.0") // Kotlin Multiplatform / Ktor 3
```

> On legacy projects still using Coil 2.x (`io.coil-kt:coil-compose`, `coil.*` packages), keep the 2.x APIs consistent or migrate via the official "Upgrading to Coil 3" guide — do not mix the two.

### 2. Primary Composable: `AsyncImage`
Use `AsyncImage` for most use cases. It handles size resolution automatically and supports standard `Image` parameters. Note `LocalPlatformContext` (from `coil3.compose`) replaces `LocalContext` for `ImageRequest.Builder`:

```kotlin
AsyncImage(
    model = ImageRequest.Builder(LocalPlatformContext.current)
        .data("https://example.com/image.jpg")
        .crossfade(true)
        .build(),
    placeholder = painterResource(R.drawable.placeholder),
    error = painterResource(R.drawable.error),
    contentDescription = stringResource(R.string.description),
    contentScale = ContentScale.Crop,
    modifier = Modifier.clip(CircleShape)
)
```

### 3. Singleton ImageLoader Configuration
Coil creates a default singleton `ImageLoader` automatically. To customize it (caches, crossfade defaults, etc.), set the factory once near your app's entry point:

```kotlin
// Compose entry point (preferred, also works in multiplatform apps)
setSingletonImageLoaderFactory { context ->
    ImageLoader.Builder(context)
        .crossfade(true)
        .build()
}
```

Alternatively, implement `SingletonImageLoader.Factory` on your `Application` class and override `newImageLoader()` (replaces Coil 2's `ImageLoaderFactory`).

### 4. Low-Level Control: `rememberAsyncImagePainter`
Use `rememberAsyncImagePainter` only when you need a `Painter` instead of a composable (e.g., for `Canvas` or `Icon`) or when you need to observe the loading state manually (`painter.state` is a `StateFlow` in Coil 3 — observe with `collectAsState()`).

> [!WARNING]
> `rememberAsyncImagePainter` does not detect the size your image is loaded at on screen and always loads the image with its original dimensions by default. Use `AsyncImage` unless a `Painter` is strictly required.

```kotlin
val painter = rememberAsyncImagePainter(
    model = ImageRequest.Builder(LocalPlatformContext.current)
        .data("https://example.com/image.jpg")
        .size(Size.ORIGINAL) // Explicitly define size if needed
        .build()
)
```

### 5. Slot API: `SubcomposeAsyncImage`
Use `SubcomposeAsyncImage` when you need a custom slot API for different states (Loading, Success, Error).

> [!CAUTION]
> Subcomposition is slower than regular composition. Avoid using `SubcomposeAsyncImage` in performance-critical areas like `LazyColumn` or `LazyRow`.

```kotlin
SubcomposeAsyncImage(
    model = "https://example.com/image.jpg",
    contentDescription = null,
    loading = {
        CircularProgressIndicator()
    },
    error = {
        Icon(Icons.Default.Error, contentDescription = null)
    }
)
```

### 6. Performance & Best Practices
*   **Singleton ImageLoader**: Use a single `ImageLoader` instance for the entire app to share the disk/memory cache (see section 3).
*   **Main-Safe**: Coil executes image requests on a background thread automatically.
*   **Crossfade**: Always enable `crossfade(true)` in `ImageRequest` for a smoother transition from placeholder to success.
*   **Sizing**: Ensure `contentScale` is set appropriately to avoid loading larger images than necessary.

### 7. Checklist for implementation
- [ ] Coil 3 coordinates (`io.coil-kt.coil3`) plus a network artifact (`coil-network-okhttp` or `coil-network-ktor3`).
- [ ] Imports from `coil3.*` packages; `LocalPlatformContext.current` for `ImageRequest.Builder`.
- [ ] Prefer `AsyncImage` over other variants.
- [ ] Always provide a meaningful `contentDescription` or set it to `null` for decorative images.
- [ ] Use `crossfade(true)` for better UX.
- [ ] Avoid `SubcomposeAsyncImage` in lists.
- [ ] Configure `ImageRequest` for specific needs like transformations (e.g., `CircleCropTransformation`).
