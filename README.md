# Android Skills

A collection of specialized skills for Android and Jetpack Compose development workflows.

Inspired by [Dimillian/Skills](https://github.com/Dimillian/Skills) — the same concept, built for Android.

## Overview

This repository contains a set of focused skills designed to assist with common Android development tasks, from generating release notes to debugging apps on emulators and maintaining code quality in Jetpack Compose.

Each skill is a self-contained `SKILL.md` file that can be loaded into an AI coding assistant (such as Claude or GitHub Copilot) to bring focused, domain-specific expertise into your session.

## Skills

### 📝 Play Store Changelog

**Purpose**: Generate user-facing Google Play Store release notes from git history.

Automatically collects commits and changes since the last git tag and transforms them into clear, benefit-focused release notes for the Play Store "What's new" section. Filters out internal-only changes and groups user-visible improvements by theme.

**Key Features**:
- Collects commits and touched files since the last tag
- Classifies changes as user-visible vs internal/tooling
- Generates concise, benefit-focused bullet points (under 500 chars)
- Validates every bullet maps back to a real commit
- Supports Fastlane metadata directory structure for multi-language notes

**Use When**: You need to write Play Store "What's new" text based on git history.

---

### 🐛 Android Emulator Debugger Agent

**Purpose**: Build, run, and debug Android projects on emulators using Gradle and ADB.

Provides a comprehensive workflow for building Android apps, launching them on emulators, interacting with the UI, and capturing logs. Handles emulator discovery, app installation, and runtime debugging.

**Key Features**:
- Discovers and manages running emulators via ADB
- Builds and installs apps with Gradle
- Interacts with UI (tap, type, swipe, key events)
- Captures and filters logcat output
- Screenshots, screen recording, and UI hierarchy inspection
- Common debugging scenarios (crashes, network, SQLite, SharedPreferences)

**Use When**: You need to run an Android app, interact with the emulator UI, inspect on-screen state, or diagnose runtime behaviour.

---

### 🔧 Compose View Refactor

**Purpose**: Refactor Jetpack Compose files for consistent structure, proper state hoisting, and clean ViewModel patterns.

Applies standardized composable ordering, UDF (Unidirectional Data Flow) patterns, and correct `@Stable` / `@Immutable` usage. Focuses on making composables lightweight, testable, and maintainable.

**Key Features**:
- Enforces consistent composable ordering (CompositionLocals → ViewModel → state → effects → UI)
- Promotes state hoisting and screen/content composable separation
- Enforces single `UiState` sealed class + separate `SharedFlow<UiEvent>` pattern
- Ensures correct `collectAsStateWithLifecycle` usage
- Applies `@Immutable` / `@Stable` to UI model classes

**Use When**: You need to clean up a composable's structure, enforce UDF patterns, or standardise ViewModel and state handling.

---

### 🚀 Compose Performance Audit

**Purpose**: Audit and improve Jetpack Compose runtime performance from code review and architecture.

Focuses on identifying recomposition hotspots, unstable types, misused `remember`, and `LazyList` pitfalls. Provides targeted refactors and guidance on using Android Studio's Layout Inspector and Composition Tracing when code review is not enough.

**Key Features**:
- Code-first review for unstable types, lambda instability, and excessive recompositions
- Targets common Compose pitfalls (`remember` misuse, state reads too high, missing `key`)
- Provides remediation guidance and refactor examples
- Guides Layout Inspector usage for recomposition count analysis
- Offers a Composition Tracing workflow for frame-level profiling

**Use When**: You need to diagnose Compose performance issues, reduce unnecessary recompositions, or get guidance on profiling with Android Studio.

---

### 🗂 Compose UI Patterns

**Purpose**: Implement common Jetpack Compose UI patterns correctly — navigation, scaffolding, bottom sheets, lists, dialogs, and pull-to-refresh.

Ensures correct API usage, proper state management for each pattern, and alignment with Material 3 conventions.

**Key Features**:
- Navigation Compose setup with typed routes and nested graphs
- `Scaffold` with scroll-aware `TopAppBar`
- `ModalBottomSheet` and `AlertDialog` patterns
- `LazyColumn` with stable keys, Paging 3, and pull-to-refresh
- Snackbar via `UiEvent` (not direct calls)
- Edge-to-edge and window insets handling

**Use When**: You need to implement standard Android UI patterns correctly, or review existing screens for API misuse.

---

### 🎨 Material 3 Theming

**Purpose**: Implement and review Jetpack Compose UI using Material 3 theming — including Dynamic Color, custom color schemes, typography, and shape systems.

Ensures correct `MaterialTheme` setup, proper token usage, Dynamic Color adoption with static fallbacks, and consistent theming across light and dark modes.

**Key Features**:
- Full `AppTheme` setup with light/dark schemes
- Dynamic Color (Android 12+, API 31) with static palette fallback
- Color token reference table and usage rules
- Typography and shape scale definitions
- Dark mode preview patterns
- Material 2 → Material 3 migration reference table

**Use When**: You need to set up or review Material 3 theming, adopt Dynamic Color, add dark mode support, or migrate from Material 2.

---

## Usage

Each skill is self-contained. To use one, point your AI coding assistant at the relevant `SKILL.md` file and ask it to follow the guidelines within.

For example, in Claude Code:
```
Read android-skills/compose-performance-audit/SKILL.md and then audit this screen for performance issues.
```

## Structure

```
android-skills/
├── play-store-changelog/
│   └── SKILL.md
├── android-emulator-debugger/
│   └── SKILL.md
├── compose-view-refactor/
│   └── SKILL.md
├── compose-performance-audit/
│   └── SKILL.md
├── compose-ui-patterns/
│   └── SKILL.md
└── material3-theming/
    └── SKILL.md
```

## Contributing

Skills are designed to be focused and reusable. When adding new skills, ensure they:
- Have a clear, single purpose
- Include a **Use When** section
- Provide concrete code examples
- Include a checklist at the end
- Follow the same structure as existing skills

## Credits

Concept inspired by [Dimillian/Skills](https://github.com/Dimillian/Skills).
