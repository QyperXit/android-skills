# Compose View Refactor

## Purpose

Refactor Jetpack Compose files for consistent structure, proper state hoisting, and clean ViewModel patterns.

Applies standardized composable ordering, UDF (Unidirectional Data Flow) patterns, and correct `@Stable` / `@Immutable` usage. Focuses on making composables lightweight, testable, and maintainable.

## Use When

- You need to clean up a composable's internal structure
- A composable is doing too much (fetching + rendering + business logic)
- State is being passed inconsistently or hoisted too low
- ViewModels are being created inside nested composables
- You need to standardise dependency injection patterns across screens

---

## Composable Structure Order

Always apply this ordering inside a composable:

```
1. CompositionLocals (via LocalX.current)
2. ViewModel (top-level screen composables only)
3. State collection (collectAsStateWithLifecycle / collectAsState)
4. Derived / remembered values (remember, derivedStateOf)
5. Side effects (LaunchedEffect, DisposableEffect, SideEffect)
6. The return / UI tree
7. Private helper composables (below the main function, same file)
```

### Example — Before

```kotlin
@Composable
fun ProfileScreen() {
    val name = remember { mutableStateOf("") }
    val vm: ProfileViewModel = hiltViewModel()
    val uiState by vm.uiState.collectAsState()
    LaunchedEffect(Unit) { vm.load() }
    Text(uiState.name)
}
```

### Example — After

```kotlin
@Composable
fun ProfileScreen(
    viewModel: ProfileViewModel = hiltViewModel()
) {
    val uiState by viewModel.uiState.collectAsStateWithLifecycle()

    LaunchedEffect(Unit) { viewModel.load() }

    ProfileContent(
        uiState = uiState,
        onRetry = viewModel::load
    )
}

@Composable
private fun ProfileContent(
    uiState: ProfileUiState,
    onRetry: () -> Unit
) {
    // Pure UI — no ViewModel, no state collection
    when (uiState) {
        is ProfileUiState.Loading -> CircularProgressIndicator()
        is ProfileUiState.Success -> Text(uiState.name)
        is ProfileUiState.Error -> RetryButton(onRetry)
    }
}
```

---

## State Hoisting Rules

- **Hoist state to the lowest common ancestor** that needs it.
- **Screen-level composables** own the ViewModel. All children receive plain data + lambdas.
- **Never pass a ViewModel** into a non-screen composable.
- **Never collect a Flow** inside a non-screen composable.
- Prefer `collectAsStateWithLifecycle()` over `collectAsState()` for lifecycle-aware collection.

```kotlin
// ❌ Wrong — ViewModel leaked into child
@Composable
fun UserCard(viewModel: ProfileViewModel) { ... }

// ✅ Correct — plain data + callback
@Composable
fun UserCard(name: String, avatarUrl: String, onTap: () -> Unit) { ... }
```

---

## ViewModel Patterns

- ViewModels expose a **single `uiState: StateFlow<UiState>`** sealed class, not multiple individual `StateFlow`s for each field.
- Side effects (navigation, toasts) are exposed via a **separate `SharedFlow<UiEvent>`**, never embedded in `UiState`.
- Use `viewModelScope` for all coroutine work.
- ViewModels must **not** hold references to `Context`, `Activity`, or any composable lambda.

```kotlin
// ✅ Preferred ViewModel structure
class ProfileViewModel @Inject constructor(
    private val repo: UserRepository
) : ViewModel() {

    private val _uiState = MutableStateFlow<ProfileUiState>(ProfileUiState.Loading)
    val uiState: StateFlow<ProfileUiState> = _uiState.asStateFlow()

    private val _events = MutableSharedFlow<ProfileEvent>()
    val events: SharedFlow<ProfileEvent> = _events.asSharedFlow()

    fun load() {
        viewModelScope.launch {
            _uiState.value = runCatching { repo.getProfile() }
                .fold(
                    onSuccess = { ProfileUiState.Success(it) },
                    onFailure = { ProfileUiState.Error(it.message) }
                )
        }
    }
}
```

---

## Stability Annotations

Apply `@Stable` or `@Immutable` to UI model classes to help the Compose compiler skip unnecessary recompositions.

| Annotation    | When to use |
|---------------|-------------|
| `@Immutable`  | All properties are `val` and their types are also deeply immutable |
| `@Stable`     | Properties may change but equality is well-defined (e.g. `data class` with mutable collections) |

```kotlin
// ✅ Correct
@Immutable
data class UserProfile(
    val id: String,
    val name: String,
    val avatarUrl: String
)
```

---

## Dependency Injection

- Use **Hilt** (`hiltViewModel()`) as the default DI approach.
- Pass dependencies into composables via **function parameters**, not `CompositionLocal` (unless it's a true ambient like theme or locale).
- Reserve `CompositionLocal` for cross-cutting concerns (analytics, theming, feature flags).

---

## Checklist Before Submitting a Refactor

- [ ] Screen composable is the only one holding a ViewModel
- [ ] All children accept plain data + lambdas
- [ ] State is collected with `collectAsStateWithLifecycle`
- [ ] UiState is a sealed class / sealed interface
- [ ] Side effects use a separate `SharedFlow<UiEvent>`
- [ ] No `Context` or `Activity` reference inside ViewModel
- [ ] Data classes passed to composables are `@Immutable` or `@Stable` where possible
- [ ] No business logic inside composable functions
