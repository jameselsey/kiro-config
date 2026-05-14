---
inclusion: fileMatch
fileMatchPattern: "**/*.gradle.kts"
---

# Android Project Standards

These standards apply to all Android projects under `jameselsey/`. They are automatically injected into any Kiro session that contains a Gradle Kotlin DSL file.

## Project conventions

- **Language**: Kotlin only. No Java.
- **Build system**: Gradle with Kotlin DSL (`build.gradle.kts`). Never Groovy.
- **Min SDK**: 24 (Android 7.0) unless there is a specific reason to go higher.
- **Target/Compile SDK**: Always the latest stable. Keep in sync — `compileSdk` and `targetSdk` should match.
- **Package naming**: `au.wombatlabs.<appname>`
- **UI**: Material Design 3 throughout. Use `?attr/color*` theme attributes, never hardcoded colours.
- **View binding**: Always enabled. Never `findViewById` in new code.
- **Dark mode**: All apps must support it. Test both themes.

## Architecture

- **Repository pattern**: All data access goes through an interface (e.g. `WantedBooksRepository`). The implementation is `Local<Name>Repository` backed by `SharedPreferences` or Room.
- **No ViewModel yet**: Activities hold state directly for now. Add ViewModel when an activity becomes complex enough to warrant it.
- **Coroutines**: Use for async work. Never `Thread { }` or `AsyncTask`.
- **No dependency injection framework**: Constructor injection only. Keep it simple.

## Freemium pattern

Every app uses the same freemium model:

- **Free tier**: Banner ads (AdMob) + feature limits
- **Premium tier**: No ads + unlimited features, unlocked via one-time in-app purchase

### AdMob

- `MobileAds.initialize()` is called in `Application.onCreate()` on the **main thread** — never wrap it in `Thread { }`.
- `AdManager` is only instantiated for free users. Check `premiumManager.shouldShowAds()` first.
- Ad container visibility: use `View.VISIBLE` / `View.GONE`, never `INVISIBLE`.

### Google Play Billing

- Product ID for the premium upgrade: `premium_upgrade` (consistent across all apps).
- `BillingManager` handles connection, purchase flow, acknowledgment, and purchase restoration.
- All billing callbacks fire on a **background thread**. Always wrap UI updates in `runOnUiThread { }`.
- `launchPurchaseFlow()` reconnects automatically if billing isn't ready — never guard the button with `isBillingAvailable()`.
- `queryPurchases()` runs on every `initialize()` call to restore premium status. It also reverts to free tier if no valid purchase is found.

### PremiumManager

- Central gate for all premium decisions. Never check `isPremium()` directly in UI code — use `PremiumManager`.
- `shouldShowAds()` → `!isPremium()`
- `canAddToWantedList(count)` → `isPremium() || count < 5`
- `getWantedListLimit()` → `null` (unlimited) for premium, `5` for free

## CI/CD

- All shared GitHub Actions live in `jameselsey/actions` (public repo). Reference them as `jameselsey/actions/<name>@main`.
- Every app repo has these workflows:
  - `build.yml` — manual trigger: unit tests + instrumented tests + debug APK
  - `release.yml` — manual trigger: semver bump (patch default) + sign + lint + GitHub Release
- Signing uses Google Play App Signing (upload key model). Keystore stored as `RELEASE_KEYSTORE` secret (base64-encoded).
- `google-services.json` is committed to private repos. It contains only Firebase project config, not secrets.

## Firebase

- Firebase Analytics is initialised in `Application.onCreate()` via `AnalyticsService.getInstance().initialize(this)`.
- `google-services.json` is committed (repos are private). Never commit it to a public repo.
- No PII in analytics events. Book titles are truncated to 100 chars.

## Secrets (GitHub Actions)

Every app repo needs these secrets configured:

| Secret | Description |
|--------|-------------|
| `RELEASE_KEYSTORE` | Base64-encoded upload keystore |
| `KEYSTORE_PASSWORD` | Keystore password |
| `KEY_ALIAS` | Key alias |
| `KEY_PASSWORD` | Key password |
| `DISCORD_WEBHOOK_URL` | Discord webhook for build failure notifications |

## String resources

- All user-facing strings in `strings.xml`. No hardcoded strings in code or layouts.
- Use `@string/` references in XML, `getString(R.string.*)` in code.

## Deprecation warnings to be aware of

- `adapterPosition` in RecyclerView — use `bindingAdapterPosition` in new code
- `setTargetResolution()` in CameraX — use `ResolutionSelector` in new code
