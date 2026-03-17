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

## Navigation (Navigation Compose)

Use **Navigation Compose** as the standard routing solution.

### Setup

```kotlin
@Composable
fun AppNavGraph(
    navController: NavHostController = rememberNavController()
) {
    NavHost(
        navController = navController,
        startDestination = "home"
    ) {
        composable("home") { HomeScreen(navController) }
        composable(
            route = "detail/{id}",
            arguments = listOf(navArgument("id") { type = NavType.StringType })
        ) { backStackEntry ->
            DetailScreen(
                id = backStackEntry.arguments?.getString("id").orEmpty(),
                navController = navController
            )
        }
    }
}
```

### Rules

- Define routes as **constants or a sealed class**, never raw strings at call sites.
- Pass `NavController` only to screen-level composables. Children navigate via **callbacks**.
- Use `SavedStateHandle` in ViewModel to read nav arguments — don't pass them directly through the call stack.
- For complex apps, scope `NavController` per nav graph and use **nested graphs**.

```kotlin
// ✅ Route constants
object Routes {
    const val HOME = "home"
    const val DETAIL = "detail/{id}"
    fun detail(id: String) = "detail/$id"
}
```

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
- [ ] Navigation routes defined as constants, not inline strings
- [ ] `NavController` not passed into non-screen composables
- [ ] Bottom sheet handles `onDismissRequest`
- [ ] `LazyColumn` items have stable `key` values
- [ ] Snackbars triggered via `UiEvent`, not called directly in composables
- [ ] `enableEdgeToEdge()` called in Activity
- [ ] Window insets applied via `Scaffold` or explicit `Modifier.windowInsetsPadding`
