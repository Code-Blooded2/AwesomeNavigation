# ⚡ Awesome Nav

A Jetpack Compose navigation library built on a fundamentally different architecture than `androidx.navigation` — both screens are alive during transitions, enabling true parallel animations, predictive back gestures, and proper per-screen lifecycle.

---

## Why?

`androidx.navigation` uses `AnimatedContent` internally. This means only **one screen is in composition at a time** — the previous screen is destroyed before the new one enters. This makes shared element transitions a workaround, predictive back gestures limited, and ViewModel teardown timing unpredictable.

Awesome Nav uses a **manual z-stack**. Every entry in the back stack is simultaneously in composition. Enter and exit screens run their animations in parallel, in true Compose fashion.

---

## How it's different

|  | Compose Navigation | Awesome Nav |
|---|---|---|
| Screens in composition during transition | 1 (sequential) | 2 (parallel) |
| Shared element transitions | Limited, workaround needed | Native — both screens alive |
| Predictive back | Partial | Full gesture control |
| Stack data structure | `ArrayDeque` + `NavGraph` + destination IDs | `List<NavEntry>` — plain Kotlin |
| ViewModel clear timing | On back stack pop | After exit animation completes |
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
        maven { url = uri("https://code-blooded2.github.io/AwesomeNavigation/awesome") }
    }
}
```

Add the dependency:

```kotlin
implementation("awesome.navigation:navigation:1.0.0")
```

---

## Setup

Initialize once in your `Application.onCreate()`:

```kotlin
class MyApp : Application() {
    override fun onCreate() {
        super.onCreate()

        AwesomeNav.setup {
            enterDuration  = 300
            exitDuration   = 250
            animationStyle = NavAnimationStyle.SlideHorizontal
            enterEasing    = FastOutSlowInEasing

            // Deep links
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
    data object Push    : NavMode()   // default — adds to stack
    data object Replace : NavMode()   // swaps current top
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

Each `NavEntry` owns its own `ViewModelStore`. ViewModels are scoped to the entry and cleared only **after the exit animation completes** — never before.

```kotlin
@Composable
fun DetailScreen(id: Int) {
    val vm: DetailViewModel = viewModel()
    // vm is cleared when this screen fully exits — not when back is pressed
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

Pass results back from screen B to screen A:

```kotlin
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

Register patterns in `AwesomeNav.setup {}`:

```kotlin
addDeepLink("myapp://detail/{id}/{title}") { params ->
    AppRoute.Detail(
        id    = params["id"]?.toIntOrNull() ?: return@addDeepLink null,
        title = params["title"] ?: return@addDeepLink null
    )
}
```

Handle in your Activity:

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

---

## Process Death

Routes are serialized automatically via `rememberSaveable`. ViewModel data is never saved — it re-fetches on restore. Make sure all route subclasses are annotated with `@Serializable`.

```kotlin
@Serializable
sealed class AppRoute : NavRoute {
    @Serializable data object Home : AppRoute()           // ✅
    @Serializable data class Detail(val id: Int) : AppRoute() // ✅
    data object Settings : AppRoute()                     // ❌ will crash
}
```

---

## Requirements

- Kotlin 2.0+
- Jetpack Compose 1.6+
- `kotlinx-serialization` plugin

---

## License

```
Copyright 2026 By CodeBlooded (Faheem)

Licensed under the Apache License, Version 2.0
```
