# Compose UI Patterns

## Purpose

Implement common Jetpack Compose UI patterns correctly — navigation, scaffolding, bottom sheets, lists, dialogs, and pull-to-refresh.

Ensures correct API usage, proper state management for each pattern, and alignment with Material 3 conventions. Covers both new implementations and refactoring legacy View-based patterns to Compose.

## Use When

- Implementing a new screen with standard navigation
- Adding a bottom sheet, dialog, or snackbar
- Building a list or feed screen
- Setting up top/bottom app bars with scroll behaviour
- Handling edge-to-edge layout and window insets

---

## Navigation (Navigation 3)

Use **Navigation 3** (`androidx.navigation3`) as the standard routing solution. Nav3 is stable as of version 1.0 and is the Compose-first replacement for Nav2.

> ⚠️ **Do not use the old `NavHost` / `NavController` Nav2 API for new screens.** Nav3 is the current standard.

### Dependency

```kotlin
implementation("androidx.navigation3:navigation3-runtime:1.0.0")
implementation("androidx.navigation3:navigation3-ui:1.0.0")
```

### Core Concepts

Nav3 replaces the hidden Nav2 back stack with a plain `SnapshotStateList`. Navigating is just adding and removing items from that list — the UI reacts automatically.

| Nav3 Concept | What it is |
|---|---|
| `NavKey` | A serializable object representing a destination |
| `rememberNavBackStack` | Creates and persists the back stack list |
| `NavDisplay` | Composable that observes the back stack and renders screens |
| `entryProvider` | Maps keys to their screen composables |

### Setup

**Step 1 — Define your destinations as keys:**

```kotlin
// Each destination is a simple serializable object or data class
@Serializable
object HomeKey : NavKey

@Serializable
data class WorkoutDetailKey(val workoutId: String) : NavKey

@Serializable
object ProfileKey : NavKey
```

**Step 2 — Create the back stack and wire up NavDisplay:**

```kotlin
@Composable
fun AppNavigation() {
    // rememberNavBackStack persists across config changes and process death
    val backStack = rememberNavBackStack(HomeKey)

    NavDisplay(
        backStack = backStack,
        entryProvider = entryProvider {
            entry<HomeKey> {
                HomeScreen(
                    onNavigateToWorkout = { id ->
                        backStack.add(WorkoutDetailKey(id))
                    }
                )
            }
            entry<WorkoutDetailKey> { key ->
                WorkoutDetailScreen(
                    workoutId = key.workoutId,
                    onBack = { backStack.removeLastOrNull() }
                )
            }
            entry<ProfileKey> {
                ProfileScreen(
                    onBack = { backStack.removeLastOrNull() }
                )
            }
        }
    )
}
```

**Step 3 — Use in Activity:**

```kotlin
override fun onCreate(savedInstanceState: Bundle?) {
    super.onCreate(savedInstanceState)
    enableEdgeToEdge()
    setContent {
        AppTheme {
            AppNavigation()
        }
    }
}
```

### Navigating

```kotlin
// Push a new destination
backStack.add(WorkoutDetailKey(workoutId = "abc123"))

// Go back
backStack.removeLastOrNull()

// Go back to a specific destination, popping everything above it
backStack.removeAll { it is WorkoutDetailKey }

// Conditional navigation (e.g. redirect to login)
if (!isLoggedIn) {
    backStack.clear()
    backStack.add(LoginKey)
}
```

### Bottom Navigation with Multiple Back Stacks

```kotlin
@Composable
fun AppNavigation() {
    val backStack = rememberNavBackStack(HomeKey)
    val currentTopKey = backStack.lastOrNull()

    Scaffold(
        bottomBar = {
            NavigationBar {
                NavigationBarItem(
                    selected = currentTopKey is HomeKey,
                    onClick = {
                        backStack.clear()
                        backStack.add(HomeKey)
                    },
                    icon = { Icon(Icons.Default.Home, null) },
                    label = { Text("Home") }
                )
                NavigationBarItem(
                    selected = currentTopKey is ProfileKey,
                    onClick = {
                        backStack.clear()
                        backStack.add(ProfileKey)
                    },
                    icon = { Icon(Icons.Default.Person, null) },
                    label = { Text("Profile") }
                )
            }
        }
    ) { innerPadding ->
        NavDisplay(
            backStack = backStack,
            modifier = Modifier.padding(innerPadding),
            entryProvider = entryProvider {
                entry<HomeKey> { HomeScreen() }
                entry<ProfileKey> { ProfileScreen() }
            }
        )
    }
}
```

### Rules

- **Never pass `backStack` into child composables** — only screen-level composables should add/remove from it. Children navigate via callbacks.
- **Keys must be `@Serializable`** if you use `rememberNavBackStack` — this is what enables persistence across process death.
- **Pass arguments via the key itself** (e.g. `WorkoutDetailKey(id)`) — not via `SavedStateHandle` or string routes.
- **ViewModels** can still read arguments via `SavedStateHandle` when Nav3 is used with Hilt.

### Migrating from Nav2

If you have existing Nav2 code (`NavHost`, `NavController`, string routes), migrate screen by screen:

| Nav2 | Nav3 equivalent |
|------|----------------|
| `NavController.navigate("detail/123")` | `backStack.add(DetailKey("123"))` |
| `NavController.popBackStack()` | `backStack.removeLastOrNull()` |
| `NavHost { composable("home") { ... } }` | `NavDisplay(entryProvider { entry<HomeKey> { ... } })` |
| `navArgument("id")` | Property on the key: `data class DetailKey(val id: String)` |
| `rememberNavController()` | `rememberNavBackStack(startKey)` |

---

## Scaffold & App Bars

Use `Scaffold` as the root of every screen. It correctly handles insets, FAB placement, and snackbar positioning.

```kotlin
@Composable
fun HomeScreen(onNavigate: (String) -> Unit) {
    val snackbarHostState = remember { SnackbarHostState() }

    Scaffold(
        topBar = {
            TopAppBar(
                title = { Text("Home") },
                colors = TopAppBarDefaults.topAppBarColors()
            )
        },
        floatingActionButton = {
            FloatingActionButton(onClick = { /* ... */ }) {
                Icon(Icons.Default.Add, contentDescription = "Add")
            }
        },
        snackbarHost = { SnackbarHost(snackbarHostState) }
    ) { innerPadding ->
        // Always consume innerPadding
        ContentList(
            modifier = Modifier.padding(innerPadding)
        )
    }
}
```

### Scroll-aware Top Bar

```kotlin
val scrollBehavior = TopAppBarDefaults.enterAlwaysScrollBehavior()

Scaffold(
    modifier = Modifier.nestedScroll(scrollBehavior.nestedScrollConnection),
    topBar = {
        TopAppBar(
            title = { Text("Feed") },
            scrollBehavior = scrollBehavior
        )
    }
) { ... }
```

---

## Bottom Sheets

Use `ModalBottomSheet` from Material 3 for action sheets and contextual panels.

```kotlin
@Composable
fun ScreenWithSheet() {
    var showSheet by remember { mutableStateOf(false) }
    val sheetState = rememberModalBottomSheetState(skipPartiallyExpanded = true)

    Button(onClick = { showSheet = true }) { Text("Open") }

    if (showSheet) {
        ModalBottomSheet(
            onDismissRequest = { showSheet = false },
            sheetState = sheetState
        ) {
            SheetContent(
                onClose = { showSheet = false }
            )
        }
    }
}
```

**Rules:**
- Always handle `onDismissRequest` — the sheet can be dismissed by swipe or back gesture.
- Set `skipPartiallyExpanded = true` unless you explicitly need a half-expanded state.
- Keep sheet content stateless where possible; hoist sheet visibility into the parent.

---

## Dialogs

Use `AlertDialog` for confirmations and `Dialog` for fully custom layouts.

```kotlin
@Composable
fun DeleteConfirmDialog(
    onConfirm: () -> Unit,
    onDismiss: () -> Unit
) {
    AlertDialog(
        onDismissRequest = onDismiss,
        title = { Text("Delete item?") },
        text = { Text("This cannot be undone.") },
        confirmButton = {
            TextButton(onClick = onConfirm) { Text("Delete") }
        },
        dismissButton = {
            TextButton(onClick = onDismiss) { Text("Cancel") }
        }
    )
}
```

---

## Lists & Feeds

### Standard List

```kotlin
LazyColumn(
    contentPadding = PaddingValues(horizontal = 16.dp, vertical = 8.dp),
    verticalArrangement = Arrangement.spacedBy(8.dp)
) {
    items(
        items = posts,
        key = { it.id },
        contentType = { "post" }
    ) { post ->
        PostCard(post = post)
    }
}
```

### Paginated List (Paging 3)

```kotlin
val pagingItems = viewModel.postsPager.collectAsLazyPagingItems()

LazyColumn {
    items(
        count = pagingItems.itemCount,
        key = pagingItems.itemKey { it.id }
    ) { index ->
        pagingItems[index]?.let { PostCard(it) }
    }

    pagingItems.apply {
        when {
            loadState.append is LoadState.Loading -> item { LoadingItem() }
            loadState.append is LoadState.Error -> item { ErrorItem(onRetry = ::retry) }
        }
    }
}
```

### Pull-to-Refresh

```kotlin
val pullRefreshState = rememberPullToRefreshState()

PullToRefreshBox(
    isRefreshing = uiState.isRefreshing,
    onRefresh = viewModel::refresh,
    state = pullRefreshState
) {
    LazyColumn { ... }
}
```

---

## Snackbars

Never show snackbars directly from composables. Emit a `UiEvent` from the ViewModel and collect it in the screen composable.

```kotlin
// In screen composable
LaunchedEffect(Unit) {
    viewModel.events.collect { event ->
        when (event) {
            is UiEvent.ShowSnackbar -> snackbarHostState.showSnackbar(event.message)
        }
    }
}
```

---

## Edge-to-Edge & Window Insets

Enable edge-to-edge in your `Activity`:

```kotlin
override fun onCreate(savedInstanceState: Bundle?) {
    super.onCreate(savedInstanceState)
    enableEdgeToEdge()
    setContent { AppTheme { AppNavGraph() } }
}
```

`Scaffold` handles most insets automatically via `innerPadding`. For custom layouts, apply insets manually:

```kotlin
Modifier.windowInsetsPadding(WindowInsets.safeDrawing)
// or for specific edges:
Modifier.padding(top = WindowInsets.statusBars.asPaddingValues().calculateTopPadding())
```

---

## Checklist

- [ ] `Scaffold` used as screen root — `innerPadding` consumed
- [ ] Using Navigation 3 (`androidx.navigation3`) not Nav2 `NavHost`
- [ ] Nav keys are `@Serializable` data classes or objects
- [ ] Arguments passed via key properties, not string routes
- [ ] `backStack` not passed into non-screen composables
- [ ] Bottom sheet handles `onDismissRequest`
- [ ] `LazyColumn` items have stable `key` values
- [ ] Snackbars triggered via `UiEvent`, not called directly in composables
- [ ] `enableEdgeToEdge()` called in Activity
- [ ] Window insets applied via `Scaffold` or explicit `Modifier.windowInsetsPadding`
