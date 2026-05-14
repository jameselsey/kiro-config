---
inclusion: fileMatch
fileMatchPattern: "**/*.gradle.kts"
---

# Android Testing Standards

These conventions apply to all Android projects. They are automatically injected into any Kiro session that contains a Gradle Kotlin DSL file.

## Test structure

| Type | Location | Runner | When to use |
|------|----------|--------|-------------|
| Unit tests | `src/test/` | JUnit4 + Robolectric | Pure logic, no emulator needed |
| Instrumented tests | `src/androidTest/` | AndroidJUnit4 on emulator | UI, SharedPreferences, Activity lifecycle |

## Unit test conventions

- Use **Robolectric** (`@RunWith(RobolectricTestRunner::class)`, `@Config(sdk = [33])`) for any unit test that needs Android Context (e.g. `SharedPreferences`, `SettingsRepository`).
- Add `testOptions { unitTests { isIncludeAndroidResources = true; isReturnDefaultValues = true } }` to `app/build.gradle.kts`.
- Use fake/stub implementations of interfaces rather than mocking frameworks. Keep fakes as inner classes in the test file.
- **JVM signature clash**: Kotlin `var` properties on classes that implement interfaces with a setter of the same name will cause a compile error. Use a private backing field with a separate setter method instead.

```kotlin
// Wrong — clashes with interface's setPremium(Boolean)
class FakePremiumRepository(var premium: Boolean = false) : PremiumRepository

// Correct
class FakePremiumRepository(isPremium: Boolean = false) : PremiumRepository {
    private var _premium = isPremium
    fun setIsPremium(value: Boolean) { _premium = value }
    override fun isPremium() = _premium
    override fun setPremium(isPremium: Boolean) { _premium = isPremium }
}
```

## Instrumented test conventions

- **Never use `isDisplayed()`** for views that may be scrolled off screen or have zero height (e.g. ad containers, scrollable help text). Use `withEffectiveVisibility(Visibility.VISIBLE/GONE)` instead — it checks the visibility flag only.
- **Seed state via SharedPreferences directly** rather than going through the billing flow or UI interactions. This makes tests fast and deterministic.
- **Wait for views before clicking**: always `check(matches(isDisplayed()))` before `perform(click())` on a FAB or button that may not be ready yet.
- **Programmatic EditText**: dialogs that build their `EditText` in code have no XML ID. Match by hint: `withHint(R.string.your_hint)`.
- Clear SharedPreferences in `@Before` and `@After` to prevent state leakage between tests.

```kotlin
@Before fun setUp() {
    context.getSharedPreferences("premium_prefs", Context.MODE_PRIVATE)
        .edit().clear().commit()
}
```

## What to test for every app

These areas must have test coverage:

- **Premium gating**: ads shown for free users, hidden for premium; feature limits enforced for free, unlimited for premium
- **Wanted list / equivalent list feature**: add/remove/retrieve, limit enforcement, `isOverLimit` for downgrade scenario
- **Matching/core algorithm**: exact match, case insensitivity, threshold enforcement, empty inputs, no false positives
- **Settings UI**: upgrade card visible for free users immediately on launch (not just after billing callback)

## CI

- Unit tests run on every build via `jameselsey/actions/android-unit-test@main` (no emulator, fast).
- Instrumented tests run via `jameselsey/actions/android-instrumented-test@main` (emulator, ~5 min).
- Both must pass before the build job runs.
- The `[EmulatorConsole]: Failed to start Emulator console for 5554` log line is a harmless warning — not a failure.

## Known gotchas

- Robolectric temp directory cleanup errors (`DirectoryNotEmptyException`) are harmless macOS filesystem warnings, not test failures.
- Billing callbacks fire on a background thread. UI assertions in instrumented tests that depend on billing state should seed SharedPreferences directly rather than triggering billing.
- `GreenThemePropertyTests` contrast check: the primary green `#308130` against near-white `#FEFFFE` achieves ~2.4:1 contrast, below WCAG AA 3:1. The test asserts ≥2.0:1 with a TODO to improve. Don't raise the threshold without updating the theme colours first.
