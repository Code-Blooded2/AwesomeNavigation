# ⚡ Awesome Nav

A Kotlin Multiplatform navigation library built on a fundamentally different architecture than `androidx.navigation` — both screens are alive during transitions, enabling true parallel animations, predictive back gestures, and proper per-screen lifecycle. Works on Android and iOS.

---

## Why?

`androidx.navigation` uses `AnimatedContent` internally. This means only **one screen is in composition at a time** — the previous screen is destroyed before the new one enters. This makes shared element transitions a workaround, predictive back gestures limited, and ViewModel teardown timing unpredictable.

Awesome Nav uses a **manual z-stack**. Only the top screen and whatever is still mid-transition beneath it are in composition — enter and exit screens run their animations in parallel, in true Compose fashion. Once a covered screen finishes its exit (or the screen above it finishes entering), it's torn out of composition and its lifecycle stops — its `ViewModelStore` is kept so state comes back intact if you navigate back to it.

---

## How it's different

|  | Compose Navigation | Awesome Nav |
|---|---|---|
| Platform | Android only | Android + iOS |
| Screens in composition during transition | 1 (sequential) | 2 (parallel) — torn down right after |
| Shared element transitions | Limited, workaround needed | Native — both screens alive |
| Predictive back | Partial | Full gesture control |
| Stack data structure | `ArrayDeque` + `NavGraph` + destination IDs | Structural-sharing stack — O(1) push/back, no full-array copy |
| ViewModel clear timing | On back stack pop | After exit animation completes |
| Screens kept alive off-screen | Entire stack, always | Only top + mid-transition entries, or when `pushForResult` asks to keep the caller alive |
| Lifecycle per screen | Shared Activity lifecycle | Per-entry `LifecycleOwner` |
| Route definition | `@Serializable` + `NavType` registration + URI conversion | Pure `@Serializable` sealed classes |
| Deep link registration | `NavDeepLink` per destination in graph | Single `addDeepLink()` pattern |
| Navigator outside Compose | Not possible without passing `NavController` | `NavRegistry.get()` anywhere |
| Framework dependency | `Fragment`, `SavedStateRegistry`, `NavGraph` | None — pure Kotlin + Compose |
| Build time overhead | Annotation processing | Zero |
| Result passing | `SavedStateHandle` workaround | First-class `StateFlow` |

---

## Installation

Add the Maven URL to your `settings.gradle.kts`:

```kotlin
dependencyResolutionManagement {
    repositories {
     maven(url = "https://code-blooded2.github.io/AwesomeNavigation/")
    }
}
```

Add the dependency:

```kotlin
// Android only
  implementation("awesome.navigation:awesome-navigation:1.0.1")

// KMP/CMP — add in commonMain
  implementation("awesome.navigation:awesome-navigation:1.0.1")
```

---

## Setup

**Android** — initialize in `Application.onCreate()`:

```kotlin
class MyApp : Application() {
    override fun onCreate() {
        super.onCreate()

        AwesomeNav.setup {
            enterDuration  = 300
            exitDuration   = 250
            animationStyle = NavAnimationStyle.SlideHorizontal
            enterEasing    = FastOutSlowInEasing

            addDeepLink("myapp://detail/{id}/{title}") { params ->
                AppRoute.Detail(
                    id    = params["id"]?.toIntOrNull() ?: return@addDeepLink null,
                    title = params["title"] ?: return@addDeepLink null
                )
            }
        }
    }
}
```

**iOS** — initialize in your Compose entry point:

```kotlin
fun MainViewController(): UIViewController = ComposeUIViewController {
    AwesomeNav.setup {
        enterDuration  = 300
        exitDuration   = 250
        animationStyle = NavAnimationStyle.SlideHorizontal

        addDeepLink("myapp://detail/{id}/{title}") { params ->
            AppRoute.Detail(
                id    = params["id"]?.toIntOrNull() ?: return@addDeepLink null,
                title = params["title"] ?: return@addDeepLink null
            )
        }
    }
    App()
}
```

---

## Quick Start

**1. Define your routes**

```kotlin
@Serializable
sealed class AppRoute : NavRoute {
    @Serializable data object Home : AppRoute()
    @Serializable data object Settings : AppRoute()
    @Serializable data class Detail(val id: Int, val title: String) : AppRoute()
}
```

**2. Set up NavHost**

```kotlin
@Composable
fun App() {
    val navigator = rememberNavigator<AppRoute>(AppRoute.Home)

    NavHost(
        navigator       = navigator,
        backgroundColor = Color.White
    ) { route ->
        when (route) {
            is AppRoute.Home     -> HomeScreen()
            is AppRoute.Settings -> SettingsScreen()
            is AppRoute.Detail   -> DetailScreen(route.id, route.title)
        }
    }
}
```

**3. Navigate**

```kotlin
// Push
navigator.push(AppRoute.Detail(id = 42, title = "Item"))

// Push, keeping the current screen composed & alive underneath
// (use this when the pushed screen will send a result back)
navigator.pushForResult(AppRoute.Detail(id = 42, title = "Item"))

// Back
navigator.back()

// Replace current
navigator.replace(AppRoute.Settings)

// Clear stack and set new root
navigator.clearAll(AppRoute.Home)

// Pop up to a route
navigator.popUpTo(AppRoute.Home, inclusive = false)
```

---

## Navigation Modes

```kotlin
sealed class NavMode {
    data object Push     : NavMode()  // default — adds to stack
    data object Replace  : NavMode()  // swaps current top
    data object ClearAll : NavMode()  // wipes entire stack, sets new root
    data class PopUpTo(
        val route: KClass<out NavRoute>,
        val inclusive: Boolean = false
    ) : NavMode()
}
```

---

## Animation Styles

```kotlin
AwesomeNav.setup {
    animationStyle = NavAnimationStyle.SlideHorizontal  // default
    animationStyle = NavAnimationStyle.SlideVertical
    animationStyle = NavAnimationStyle.Fade
    animationStyle = NavAnimationStyle.ScaleFade
    animationStyle = NavAnimationStyle.None
}
```

---

## ViewModel Scoping

Each `NavEntry` owns its own `ViewModelStore`. The store is only **cleared** — actually destroyed — when the entry is popped off the stack for good, after its exit animation completes. Never before, and never just because it was pressed back on.

Being **covered** by a new screen is different from being **removed**: once the screen on top finishes entering, the covered screen is torn out of composition (its lifecycle stops) to free memory, but its `ViewModelStore` is retained. Navigate back to it and the same ViewModel instance comes back with its state intact. Push it with `pushForResult` instead, and it stays composed and resumed the whole time the screen above it is up.

```kotlin
@Composable
fun DetailScreen(id: Int) {
    val vm: DetailViewModel = viewModel()
    // vm survives being covered by another screen — it's only cleared
    // once this entry is actually popped off the stack and exits
}
```

---

## Per-Screen Lifecycle

Every entry has its own `LifecycleOwner`. `ON_RESUME` fires only on the top screen. Background screens receive `ON_PAUSE`.

```kotlin
class DetailViewModel : ViewModel() {
    init {
        viewModelScope.launch {
            lifecycle.repeatOnLifecycle(Lifecycle.State.RESUMED) {
                // only runs when this screen is on top
            }
        }
    }
}
```

---

## Result Passing

Pass results back from screen B to screen A. Push B with `pushForResult` so screen A stays composed and resumed-underneath instead of being torn down while B is on top:

```kotlin
// Screen A — push for a result so A isn't destroyed while B is up
navigator.pushForResult(AppRoute.Picker)

// Screen B — set result before going back
navigator.setResult("picked_item", selectedItem)
navigator.back()

// Screen A — observe and consume
LaunchedEffect(Unit) {
    navigator.results.collect {
        val item = navigator.consumeTypedResult<Item>("picked_item") ?: return@collect
        // handle result
    }
}
```

---

## Access Navigator Anywhere

```kotlin
// From a ViewModel, Service, or BroadcastReceiver
val navigator = NavRegistry.get<AppRoute>()
navigator?.push(AppRoute.Detail(id = 1, title = "From ViewModel"))
```

For multiple `NavHost` instances, use keys:

```kotlin
NavHost(navigator = navigator, key = "main") { ... }

// Access by key
NavRegistry.get<AppRoute>("main")
```

---

## Deep Links

Register patterns in `AwesomeNav.setup {}` — works on both platforms:

```kotlin
addDeepLink("myapp://detail/{id}/{title}") { params ->
    AppRoute.Detail(
        id    = params["id"]?.toIntOrNull() ?: return@addDeepLink null,
        title = params["title"] ?: return@addDeepLink null
    )
}
```

**Android** — handle in Activity:

```kotlin
override fun onCreate(savedInstanceState: Bundle?) {
    super.onCreate(savedInstanceState)
    setContent { App() }
    intent?.let { AwesomeNav.handleDeepLink(it) }
}

override fun onNewIntent(intent: Intent) {
    super.onNewIntent(intent)
    AwesomeNav.handleDeepLink(intent)
}
```

Add intent filter in `AndroidManifest.xml`:

```xml
<intent-filter>
    <action android:name="android.intent.action.VIEW" />
    <category android:name="android.intent.category.DEFAULT" />
    <category android:name="android.intent.category.BROWSABLE" />
    <data android:scheme="myapp" />
</intent-filter>
```

**iOS** — handle in AppDelegate:

```swift
func application(
    _ app: UIApplication,
    open url: URL,
    options: [UIApplication.OpenURLOptionsKey: Any] = [:]
) -> Bool {
    return AwesomeNav.shared.handleDeepLink(url: url)
}
```

**Common** — pass URI string directly on any platform:

```kotlin
AwesomeNav.handleDeepLink("myapp://detail/42/title")
```

---

## Process Death

Routes are serialized automatically via `rememberSaveable`. ViewModel data is never saved — it re-fetches on restore. Make sure all route subclasses are annotated with `@Serializable`.

```kotlin
@Serializable
sealed class AppRoute : NavRoute {
    @Serializable data object Home : AppRoute()                // ✅
    @Serializable data class Detail(val id: Int) : AppRoute() // ✅
    data object Settings : AppRoute()                          // ❌ will crash
}
```

---

## Requirements

- Kotlin 2.0+
- Compose Multiplatform 1.8+
- `kotlinx-serialization` plugin
- Android Min SDK 26
- iOS 16+

---

## Roadmap

- [ ] Shared element transitions
- [ ] Predictive back gesture
- [ ] Bottom nav / multi-backstack
- [ ] Dialog and bottom sheet destinations

---

## License

```
Copyright 2026 By CodeBlooded (Faheem)

Licensed under the Apache License, Version 2.0
```
