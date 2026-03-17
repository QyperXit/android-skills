# Material 3 Theming

## Purpose

Implement and review Jetpack Compose UI using Material 3 theming — including Dynamic Color, custom color schemes, typography, and shape systems.

Ensures correct `MaterialTheme` setup, proper token usage, Dynamic Color adoption with fallbacks, and consistent theming across light/dark modes.

## Use When

- Setting up Material 3 theming in a new project
- Migrating from Material 2 (`androidx.compose.material`) to Material 3 (`androidx.compose.material3`)
- Adopting Dynamic Color (Android 12+) with a static fallback palette
- Reviewing composables for hardcoded colors or incorrect token usage
- Adding dark mode support

---

## Theme Setup

The canonical `AppTheme` wrapper. Always provide both light and dark schemes, and opt into Dynamic Color where supported.

```kotlin
private val LightColorScheme = lightColorScheme(
    primary = Brand500,
    onPrimary = Color.White,
    primaryContainer = Brand100,
    onPrimaryContainer = Brand900,
    secondary = Teal400,
    onSecondary = Color.White,
    background = Color(0xFFFFFBFE),
    surface = Color(0xFFFFFBFE),
    // ... define all required tokens
)

private val DarkColorScheme = darkColorScheme(
    primary = Brand200,
    onPrimary = Brand800,
    primaryContainer = Brand700,
    onPrimaryContainer = Brand100,
    secondary = Teal200,
    onSecondary = Teal800,
    background = Color(0xFF1C1B1F),
    surface = Color(0xFF1C1B1F),
    // ...
)

@Composable
fun AppTheme(
    darkTheme: Boolean = isSystemInDarkTheme(),
    dynamicColor: Boolean = true,
    content: @Composable () -> Unit
) {
    val colorScheme = when {
        dynamicColor && Build.VERSION.SDK_INT >= Build.VERSION_CODES.S -> {
            val context = LocalContext.current
            if (darkTheme) dynamicDarkColorScheme(context)
            else dynamicLightColorScheme(context)
        }
        darkTheme -> DarkColorScheme
        else -> LightColorScheme
    }

    MaterialTheme(
        colorScheme = colorScheme,
        typography = AppTypography,
        shapes = AppShapes,
        content = content
    )
}
```

---

## Color Tokens

Always reference colors via `MaterialTheme.colorScheme` tokens. **Never hardcode `Color(...)` values directly in composables.**

### Token Reference

| Token | Typical Use |
|-------|-------------|
| `primary` | Key action buttons, FABs, active indicators |
| `onPrimary` | Text/icons on primary-colored surfaces |
| `primaryContainer` | Tonal button backgrounds, chips |
| `onPrimaryContainer` | Text/icons inside primary containers |
| `secondary` | Secondary actions, filters |
| `tertiary` | Complementary accent |
| `surface` | Card and sheet backgrounds |
| `surfaceVariant` | Input field fills, tonal surfaces |
| `outline` | Borders, dividers |
| `error` / `onError` | Validation, destructive actions |

```kotlin
// ❌ Wrong — hardcoded color, breaks dark mode and Dynamic Color
Text("Hello", color = Color(0xFF6200EE))

// ✅ Correct — respects theme
Text("Hello", color = MaterialTheme.colorScheme.primary)
```

---

## Typography

Define a custom `Typography` object using `TextStyle` with your chosen font family.

```kotlin
val AppTypography = Typography(
    displayLarge = TextStyle(
        fontFamily = BrandFontFamily,
        fontWeight = FontWeight.Normal,
        fontSize = 57.sp,
        lineHeight = 64.sp,
        letterSpacing = (-0.25).sp
    ),
    titleLarge = TextStyle(
        fontFamily = BrandFontFamily,
        fontWeight = FontWeight.SemiBold,
        fontSize = 22.sp,
        lineHeight = 28.sp
    ),
    bodyMedium = TextStyle(
        fontFamily = BrandFontFamily,
        fontWeight = FontWeight.Normal,
        fontSize = 14.sp,
        lineHeight = 20.sp,
        letterSpacing = 0.25.sp
    ),
    // ... define all roles you use
)
```

Reference in composables:

```kotlin
// ✅ Always use theme typography roles
Text("Headline", style = MaterialTheme.typography.headlineMedium)
Text("Body copy", style = MaterialTheme.typography.bodyMedium)
```

---

## Shapes

Material 3 uses a shape scale from `None` → `ExtraSmall` → `Small` → `Medium` → `Large` → `ExtraLarge` → `Full`.

```kotlin
val AppShapes = Shapes(
    extraSmall = RoundedCornerShape(4.dp),   // chips, tooltips
    small = RoundedCornerShape(8.dp),         // text fields, small buttons
    medium = RoundedCornerShape(12.dp),       // cards, menus
    large = RoundedCornerShape(16.dp),        // bottom sheets, side sheets
    extraLarge = RoundedCornerShape(28.dp)    // dialogs, large surfaces
)
```

Reference in composables via `MaterialTheme.shapes`:

```kotlin
// ✅ Consistent with theme shape scale
Card(shape = MaterialTheme.shapes.medium) { ... }
```

---

## Dynamic Color

Dynamic Color (Android 12+, API 31) extracts a palette from the user's wallpaper. Always provide a static fallback.

```kotlin
// AppTheme already handles this — verify your theme wrapper follows the pattern above.
// Key rules:
// 1. Gate on Build.VERSION.SDK_INT >= Build.VERSION_CODES.S
// 2. Use dynamicLightColorScheme(context) / dynamicDarkColorScheme(context)
// 3. Fall back to your static LightColorScheme / DarkColorScheme
// 4. Never hardcode colors inside components — they must read from MaterialTheme.colorScheme
```

---

## Dark Mode

- Use `isSystemInDarkTheme()` as the default. Allow override via your `AppTheme(darkTheme = ...)` parameter for previews and testing.
- All icons should use `LocalContentColor.current` or explicit `MaterialTheme.colorScheme` tokens so they adapt automatically.
- Test both modes in `@Preview`:

```kotlin
@Preview(uiMode = Configuration.UI_MODE_NIGHT_NO, name = "Light")
@Preview(uiMode = Configuration.UI_MODE_NIGHT_YES, name = "Dark")
@Composable
private fun MyComponentPreview() {
    AppTheme { MyComponent() }
}
```

---

## Surface Elevation & Tonal Elevation

In Material 3, elevation expresses hierarchy via **tonal colour** (not just shadow). Higher surfaces get a tinted overlay in dark mode.

```kotlin
// ✅ Use Surface/Card with elevation — tonal colour applied automatically
Surface(
    tonalElevation = 2.dp,
    shape = MaterialTheme.shapes.medium
) {
    content()
}
```

Avoid manually painting elevation shadows. Rely on Material 3 components (`Card`, `Surface`, `NavigationBar`) to handle this correctly.

---

## Migration from Material 2

| Material 2 | Material 3 Equivalent |
|------------|----------------------|
| `MaterialTheme.colors.primary` | `MaterialTheme.colorScheme.primary` |
| `MaterialTheme.typography.h6` | `MaterialTheme.typography.titleLarge` |
| `MaterialTheme.shapes.medium` | `MaterialTheme.shapes.medium` (same) |
| `Surface(elevation = X)` | `Surface(tonalElevation = X)` |
| `TopAppBar` | `TopAppBar` from `material3` package |
| `BottomNavigation` | `NavigationBar` |
| `Scaffold` | `Scaffold` from `material3` package |

**Never mix `androidx.compose.material` and `androidx.compose.material3` imports in the same file.** It causes visual inconsistency and conflicts.

---

## Checklist

- [ ] `AppTheme` wraps all composables in previews and in `Activity`
- [ ] Dynamic Color gated on API 31+ with static fallback
- [ ] No hardcoded `Color(...)` values inside composable functions
- [ ] All text uses `MaterialTheme.typography` roles
- [ ] All shapes use `MaterialTheme.shapes` scale
- [ ] Both light and dark `@Preview` annotations on UI components
- [ ] No mixed `material` + `material3` imports in the same file
- [ ] `enableEdgeToEdge()` called in `Activity.onCreate` alongside theming
