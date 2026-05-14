---
inclusion: manual
---

# New Android App — Setup SOP

Pull this into a session with `#android-new-app` when starting a new Android project.

## Before you start

- [ ] App idea is defined — name, package name (`au.wombatlabs.<name>`), one-line description
- [ ] New private GitHub repo created: `jameselsey/<appname>`
- [ ] Android Studio project created with Kotlin DSL, min SDK 24, target SDK 35

## Step 1: Project scaffold

Apply these to the new project immediately after creation:

- Enable view binding in `app/build.gradle.kts`: `buildFeatures { viewBinding = true }`
- Add `testOptions { unitTests { isIncludeAndroidResources = true; isReturnDefaultValues = true } }`
- Add test dependencies: `testImplementation("org.robolectric:robolectric:4.12.2")`, `testImplementation("androidx.test:core:1.5.0")`, `testImplementation("androidx.test.ext:junit:1.1.5")`
- Set `versionCode` and `versionName` from env vars (same pattern as goodfinds)
- Add signing config reading from env vars

## Step 2: Firebase

1. Create a new Firebase project at console.firebase.google.com
2. Add Android app with the correct package name
3. Download `google-services.json` → commit to repo (private repo, safe to commit)
4. Add `com.google.gms.google-services` plugin to both `build.gradle.kts` files
5. Add Firebase BOM + `firebase-analytics-ktx` dependency
6. Create `FindrApplication`-equivalent class, call `AnalyticsService.getInstance().initialize(this)`
7. Register in `AndroidManifest.xml` with `android:name`

## Step 3: AdMob

1. Create AdMob account / add app at admob.google.com
2. Register the app, get App ID (`ca-app-pub-XXXX~YYYY`)
3. Create a Banner ad unit, get Ad Unit ID (`ca-app-pub-XXXX/ZZZZ`)
4. Add App ID to `AndroidManifest.xml` as `com.google.android.gms.ads.APPLICATION_ID` meta-data
5. Add `com.google.android.gms:play-services-ads` dependency
6. Copy `AdManager.kt` from goodfinds, update `AD_UNIT_ID`
7. Call `MobileAds.initialize(this)` in `Application.onCreate()` on the **main thread**

## Step 4: Freemium (AdMob + Play Billing)

Copy these files from goodfinds and update package names:
- `data/PremiumRepository.kt` (interface)
- `data/LocalPremiumRepository.kt`
- `service/PremiumManager.kt`
- `service/BillingManager.kt`
- `service/AdManager.kt`

In Play Console (after first upload):
- Monetize → Products → In-app products → Create product
- Product ID: `premium_upgrade`
- Type: One-time product
- Activate it

## Step 5: GitHub Actions

Copy from goodfinds:
- `.github/workflows/build.yml` (manual trigger)
- `.github/workflows/release.yml` (semver bump)
- `.github/workflows/weekly-digest.yml`
- `.github/workflows/api-level-monitor.yml`
- `.github/workflows/config/monitoring-config.json` (update SDK versions and deadline)

All workflows reference `jameselsey/actions` — no changes needed there.

## Step 6: GitHub Secrets

Add these to the new repo (Settings → Secrets and variables → Actions):

| Secret | How to get it |
|--------|---------------|
| `RELEASE_KEYSTORE` | `keytool -genkey ...` then `base64 release.keystore` |
| `KEYSTORE_PASSWORD` | Password you chose |
| `KEY_ALIAS` | Alias you chose |
| `KEY_PASSWORD` | Password you chose |
| `DISCORD_WEBHOOK_URL` | Discord server → channel settings → Integrations → Webhooks |

## Step 7: Play Console setup

1. Upload first AAB to internal testing track (required before billing API works)
2. Create `premium_upgrade` in-app product (see Step 4)
3. Add your Google account as a licence tester: Setup → Licence testing
4. Set up Play App Signing when prompted on first upload

## Step 8: Monitoring config

Update `.github/workflows/config/monitoring-config.json`:
- `api_requirements.next_target_sdk` — current Google Play requirement
- `api_requirements.next_deadline` — deadline date from Play Console policy page
- `notifications.webhook_secret_name` — `DISCORD_WEBHOOK_URL`

## Step 9: First tests

Write these before shipping:
- `PremiumManagerTest` — ad gating, list limits (copy from goodfinds, no changes needed)
- `<Feature>RepositoryTest` — the app's equivalent of WantedBooksRepositoryTest
- `MatchingServiceTest` or equivalent core algorithm test
- `AdVisibilityInstrumentedTest` — copy from goodfinds, update activity names
- `SettingsPremiumUIInstrumentedTest` — copy from goodfinds, no changes needed

## Step 10: docs/

Create these in `docs/`:
- `release.md`
- `monitoring.md`
- `firebase.md`
- `monetization.md`
- `play-store.md`
- `privacy-policy.md`

Keep `README.md` brief — one paragraph, tech stack, build command, link table to docs/.
