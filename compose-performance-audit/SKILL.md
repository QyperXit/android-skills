# Compose Performance Audit

## Purpose

Audit and improve Jetpack Compose runtime performance from code review and architecture.

Focuses on identifying recomposition hotspots, misuse of `remember`, unstable types, and `LazyList` pitfalls. Provides targeted refactors and guidance on using Android Studio's Layout Inspector and Composition Tracing when code review alone is not enough.

## Use When

- Scrolling feels janky or lists stutter
- You suspect excessive recompositions
- Animations are dropping frames
- A screen feels slow to render on first load
- You want to validate a composable is skippable before shipping

---

## Step 1 — Code Review (Start Here)

Work through these categories in order before reaching for profiling tools.

### 1.1 Unstable Types Causing Recomposition

The Compose compiler will **not skip** recomposition of a composable if any of its parameters are considered unstable.

A type is **unstable** if:
- It is an interface (unless annotated)
- It contains a `var` property
- It uses a `List`, `Map`, or `Set` from `kotlin.collections` (mutable by default in the compiler's view)

**Fix:** Use `@Immutable` / `@Stable`, or replace `List<T>` with `kotlinx.collections.immutable.ImmutableList<T>`.

```kotlin
// ❌ Unstable — List<T> is not guaranteed immutable
@Composable
fun TagRow(tags: List<String>) { ... }

// ✅ Stable — compiler can skip recomposition
@Composable
fun TagRow(tags: ImmutableList<String>) { ... }
```

### 1.2 Lambda Instability

Anonymous lambdas passed to composables are recreated on every recomposition, causing children to recompose even when nothing changed.

**Fix:** Use `remember` to stabilise lambdas, or define them as named references.

```kotlin
// ❌ New lambda instance every recomposition
ItemList(items = items, onTap = { viewModel.select(it) })

// ✅ Stable reference
val onTap = remember { { item: Item -> viewModel.select(item) } }
ItemList(items = items, onTap = onTap)
```

### 1.3 Unnecessary `remember` / Missing `remember`

| Mistake | Fix |
|---------|-----|
| Heavy object created without `remember` | Wrap in `remember { }` |
| `derivedStateOf` used without `remember` | Always `remember { derivedStateOf { } }` |
| `remember` used for a value that doesn't need caching | Remove it |
| `rememberSaveable` missing for state that should survive rotation | Replace `remember` with `rememberSaveable` |

```kotlin
// ❌ DateTimeFormatter recreated every recomposition
val formatter = DateTimeFormatter.ofPattern("dd MMM yyyy")

// ✅ Cached
val formatter = remember { DateTimeFormatter.ofPattern("dd MMM yyyy") }
```

### 1.4 Reading State Too High

Reading a `State<T>` value at a high level in the tree causes all descendants to recompose when it changes.

**Fix:** Defer state reads as deep as possible, or pass lambdas instead of values.

```kotlin
// ❌ offset read here triggers full recomposition of HeavyScreen
@Composable
fun HeavyScreen(scrollState: ScrollState) {
    val offset = scrollState.value  // read too high
    Box(modifier = Modifier.offset(y = offset.dp)) { ... }
}

// ✅ Defer read into modifier lambda
@Composable
fun HeavyScreen(scrollState: ScrollState) {
    Box(modifier = Modifier.offset { IntOffset(0, scrollState.value) }) { ... }
}
```

### 1.5 LazyList Pitfalls

- **Missing `key`**: Without stable keys, Compose cannot diff the list correctly and will recompose all visible items.
- **Heavy `item` blocks**: Each `item { }` lambda should be as lightweight as possible. Extract named composables.
- **`contentType`**: Set `contentType` for heterogeneous lists to improve item reuse.

```kotlin
// ❌ No key, no contentType
LazyColumn {
    items(posts) { post -> PostCard(post) }
}

// ✅ Keys + contentType
LazyColumn {
    items(
        items = posts,
        key = { it.id },
        contentType = { it::class }
    ) { post ->
        PostCard(post)
    }
}
```

---

## Step 2 — Layout Inspector (Recomposition Counts)

When code review doesn't reveal the problem:

1. Run app on emulator or device (debug build)
2. Open **Android Studio → View → Tool Windows → Layout Inspector**
3. Enable **"Show recomposition counts"**
4. Interact with the screen
5. Look for composables with high or rapidly increasing recomposition counts
6. Cross-reference with Step 1 fixes

---

## Step 3 — Composition Tracing (Frame-level Profiling)

For dropped frames and jank that aren't explained by recomposition counts:

1. Add the tracing dependency:
   ```kotlin
   implementation("androidx.compose.runtime:runtime-tracing:1.0.0")
   ```
2. Run a **CPU profile** in Android Studio Profiler using the **"System Trace"** preset
3. Look for `Choreographer#doFrame` taking >16ms
4. Expand compose frames to see which composable is the bottleneck
5. Apply targeted fixes from Step 1

---

## Common Fixes Reference

| Symptom | Likely Cause | Fix |
|---------|-------------|-----|
| Everything recomposes on scroll | State read too high | Defer state reads |
| List items flicker on update | Missing `key` | Add stable `key` to `items()` |
| Slow first render | Heavy work in `body` | Move to `LaunchedEffect` or ViewModel |
| Animation jank | Unstable lambda in animated composable | `remember` the lambda |
| All items recompose on single item change | Unstable list type | Use `ImmutableList` |
| State survives process death but not rotation | `remember` instead of `rememberSaveable` | Replace with `rememberSaveable` |

---

## Checklist Before Shipping

- [ ] All data classes passed as parameters are `@Immutable` or `@Stable`
- [ ] `List<T>` parameters replaced with `ImmutableList<T>` where needed
- [ ] Lambdas in hot paths are stabilised with `remember`
- [ ] `LazyColumn` / `LazyRow` items have stable `key` values
- [ ] Heavy computations are inside `remember { }` blocks
- [ ] `derivedStateOf` is always wrapped in `remember { }`
- [ ] State reads are deferred as deep into the tree as possible
- [ ] No disk/network I/O inside composable functions
- [ ] Layout Inspector shows no unexpected recomposition spikes
