# Build & Deploy — EAS, OTA, Monitoring (June 2026)

## EAS — PRODUCTION STANDARD

EAS = cloud native builds, OTA updates, store submission, credentials vault.
Android CI without a Mac. iOS via EAS cloud (Xcode 26.4+ images for SDK 56).

**SDK 56:** Hermes bytecode diffing **on by default** for `expo-updates` (~58% smaller OTA patches avg).

Opt out: `"enableBsdiffPatchSupport": false` in `app.json` `updates` block.

---

## INITIAL EAS SETUP

```bash
# Install EAS CLI
npm install -g eas-cli

# Login to Expo account
eas login

# Initialize EAS in your project
eas init

# Configure build profiles
eas build:configure
```

This creates `eas.json`:

```json
{
  "cli": {
    "version": ">= 12.0.0"
  },
  "build": {
    "development": {
      "developmentClient": true,
      "distribution": "internal",
      "ios": {
        "simulator": true
      }
    },
    "preview": {
      "distribution": "internal",
      "android": {
        "buildType": "apk"
      }
    },
    "production": {
      "autoIncrement": true
    }
  },
  "submit": {
    "production": {}
  }
}
```

---

## BUILD PROFILES EXPLAINED

| Profile | Use for | Distribution |
|---|---|---|
| `development` | Dev builds with dev client | Internal (your devices) |
| `preview` | Testing before release, share with testers | Internal (APK/IPA) |
| `production` | App Store / Play Store submission | Store |

```bash
# Build for development (installs on your device)
eas build --platform all --profile development

# Build for internal testing (share link)
eas build --platform all --profile preview

# Build for stores
eas build --platform all --profile production
```

---

## OTA UPDATES — THE SUPERPOWER

Push JS/assets updates without App Store review.
**Hermes bytecode diffing default SDK 56** — binary patches vs full bundles on EAS Update.

```bash
# Push update to all users on production channel
eas update --channel production --message "Fix login bug"

# Push update to specific branch
eas update --branch main --message "Update UI"
```

### Update strategy in app.json
```json
{
  "expo": {
    "updates": {
      "enabled": true,
      "checkAutomatically": "ON_LOAD",
      "fallbackToCacheTimeout": 0
    }
  }
}
```

### Manual update check (for critical updates)
```tsx
import * as Updates from 'expo-updates';

async function checkForUpdate() {
  try {
    const update = await Updates.checkForUpdateAsync();
    if (update.isAvailable) {
      await Updates.fetchUpdateAsync();
      await Updates.reloadAsync();  // Restarts app with new bundle
    }
  } catch (error) {
    console.error('Update check failed:', error);
  }
}
```

**What OTA can update:** JS code, assets, images, fonts.
**What OTA cannot update:** native code, config plugins, app.json native changes.
Anything that changes native → requires a new store build.

---

## GRADLE CACHE (faster Android EAS builds)

Enable in `.env`:
```
EAS_GRADLE_CACHE=1
```

Or in `eas.json`:
```json
{
  "build": {
    "production": {
      "env": {
        "EAS_GRADLE_CACHE": "1"
      }
    }
  }
}
```

Always enable for Android EAS builds.

---

## COMPILER CACHE (ccache — up to ~30% on cache hit)

Expo changelog: **ccache** for C/C++ recompilation on EAS Build (SDK 53+ Android, SDK 54+ iOS).

Set in EAS project env (all profiles: development, preview, production):

```
EAS_USE_CACHE=1
```

Optional fine-grain: `EAS_RESTORE_CACHE=1`, `EAS_SAVE_CACHE=1`.

Custom workflows: `eas/restore_build_cache` before compile, `eas/save_build_cache` after.

Docs: https://expo.dev/changelog/compiler-cache-for-builds

**Distinct from** `EAS_GRADLE_CACHE` — use **both** when available.

---

## LOCAL RUN CACHE (experimental)

Speed up `npx expo run:ios` / `run:android` by fingerprint-matched remote binaries:

```powershell
npm install --save-dev eas-build-cache-provider
```

```json
{
  "expo": {
    "experiments": {
      "buildCacheProvider": "eas"
    }
  }
}
```

Simulator builds only for iOS cache reuse (not physical devices). See https://docs.expo.dev/guides/cache-builds-remotely

---

## EAS WORKFLOWS (CI orchestration)

YAML in `.eas/workflows/*.yml` — build, submit, update, Maestro, Slack in one pipeline.

```yaml
# .eas/workflows/nightly-maestro.yml
name: Nightly smoke

on:
  schedule:
    - cron: "0 6 * * 1-5"  # Weekdays 06:00 GMT

jobs:
  build_ios:
    type: build
    params:
      platform: ios
      profile: preview
```

**Cron rules:** default branch only · **GMT** timezone · design idempotent jobs (may delay/skip at hour boundaries).

Trigger manually anytime: `eas workflow:run`.

Docs: https://docs.expo.dev/eas/workflows/introduction

---

## CMake PRECOMPILED HEADERS (SDK 56, faster debug builds)

In `app.json`:
```json
{
  "expo": {
    "plugins": [
      ["expo-build-properties", {
        "android": {
          "usePrecompiledHeaders": true
        }
      }]
    ]
  }
}
```

Android CMake debug builds: 2.81x faster. Enable it.

---

## APP SIGNING

### Android
```bash
# EAS manages keystore automatically
# For existing keystores:
eas credentials --platform android
```

### iOS
```bash
# EAS manages provisioning profiles + certificates
eas credentials --platform ios

# Let EAS handle everything (recommended)
# In eas.json:
{
  "build": {
    "production": {
      "ios": {
        "credentialsSource": "remote"
      }
    }
  }
}
```

EAS stores credentials securely in their vault. No local keystore management.

---

## APP SUBMISSION

```bash
# Submit latest build to App Store + Play Store
eas submit --platform all

# Submit specific build
eas submit --platform ios --id [build-id]
```

Prerequisites:
- App Store: Apple Developer account ($99/year), app created in App Store Connect
- Play Store: Google Play Developer account ($25 one-time), app created in Play Console

---

## ENVIRONMENT SECRETS (EAS)

For server-only secrets (API keys that should never be in the JS bundle):

```bash
eas secret:create --scope project --name MY_SECRET_KEY --value "secret-value"
```

Access in EAS build hooks and config plugins — not in JS code.

For client-side config (safe to be in bundle):
```
EXPO_PUBLIC_API_URL=https://api.myapp.com
```

---

## REVENUE & MONETIZATION — REVENUECATAT

```bash
npm install react-native-purchases
npx expo install react-native-purchases
```

```tsx
import Purchases, { LOG_LEVEL } from 'react-native-purchases';
import { Platform } from 'react-native';

// Initialize (call early in app lifecycle)
async function initPurchases() {
  if (__DEV__) {
    Purchases.setLogLevel(LOG_LEVEL.DEBUG);
  }

  await Purchases.configure({
    apiKey: Platform.OS === 'ios'
      ? process.env.EXPO_PUBLIC_RC_IOS_KEY!
      : process.env.EXPO_PUBLIC_RC_ANDROID_KEY!,
  });
}

// Check if user is subscribed
async function checkSubscription() {
  const customerInfo = await Purchases.getCustomerInfo();
  return customerInfo.entitlements.active['premium'] !== undefined;
}

// Show paywall
async function purchase() {
  const offerings = await Purchases.getOfferings();
  const monthly = offerings.current?.monthly;
  if (!monthly) return;

  const { customerInfo } = await Purchases.purchasePackage(monthly);
  if (customerInfo.entitlements.active['premium']) {
    // Unlock premium features
  }
}
```

RevenueCat works with both App Store and Play Store. Single API for both platforms.
For regions without native payment (like Iraq), use their webhook system to integrate
Zain Cash / FastPay payments that update entitlements via RevenueCat API.

---

## SENTRY SETUP

```bash
npx expo install @sentry/react-native
npx sentry-wizard@latest -i reactNative
```

```tsx
// app/_layout.tsx
import * as Sentry from '@sentry/react-native';

const useLegacyRnFetch = process.env.EXPO_PUBLIC_USE_RN_FETCH === '1';

Sentry.init({
  dsn: process.env.EXPO_PUBLIC_SENTRY_DSN,
  environment: __DEV__ ? 'development' : 'production',
  tracesSampleRate: 0.2,
  profilesSampleRate: 0.2,
  integrations: [
    // traceFetch only for expo/fetch — disable when EXPO_PUBLIC_USE_RN_FETCH=1
    Sentry.reactNativeTracingIntegration({ traceFetch: !useLegacyRnFetch }),
    Sentry.breadcrumbsIntegration({ fetch: true }),
  ],
});

export default Sentry.wrap(function RootLayout() {
  // ...
});
```

Sentry automatically:
- Captures JS crashes with sourcemaps (readable stack traces)
- Captures native crashes (iOS/Android)
- Tracks performance (slow renders, network calls)
- Works with EAS — upload sourcemaps automatically

---

## POSTHOG SETUP

```bash
npm install posthog-react-native
npx expo install expo-file-system expo-application
```

```tsx
// app/_layout.tsx
import PostHog, { PostHogProvider } from 'posthog-react-native';

const posthog = new PostHog(process.env.EXPO_PUBLIC_POSTHOG_KEY!, {
  host: 'https://app.posthog.com',
});

export default function RootLayout() {
  return (
    <PostHogProvider client={posthog}>
      {/* rest of app */}
    </PostHogProvider>
  );
}

// Usage
import { usePostHog } from 'posthog-react-native';

function FeatureScreen() {
  const posthog = usePostHog();

  function handleAction() {
    posthog.capture('feature_used', { feature: 'export', format: 'pdf' });
  }
}
```

---

## VERSIONING STRATEGY

In `app.json`:
```json
{
  "expo": {
    "version": "1.2.0",          ← User-visible version (semver)
    "ios": {
      "buildNumber": "42"        ← Auto-incremented by EAS
    },
    "android": {
      "versionCode": 42          ← Auto-incremented by EAS
    }
  }
}
```

With `"autoIncrement": true` in eas.json production profile:
EAS automatically bumps buildNumber / versionCode on every build.
You only manually bump `version` when releasing user-visible changes.

---

## PRE-SUBMISSION CHECKLIST

```
Technical:
- [ ] All console.logs removed (or metro config drops them)
- [ ] No __DEV__ code reaching production
- [ ] Environment variables set in EAS Secrets + .env.production
- [ ] Sentry DSN configured
- [ ] OTA update channel set to 'production'
- [ ] Icon and splash screen in all required sizes
- [ ] iOS: privacy descriptions in app.json for camera, location, etc.
- [ ] Android: permissions in app.json
- [ ] Accessibility: all interactive elements labeled
- [ ] Tested on real iOS device + real low-end Android

App Store specific:
- [ ] App Privacy details filled in App Store Connect
- [ ] Age rating set correctly
- [ ] Screenshots for all required device sizes
- [ ] App description + keywords optimized
- [ ] No placeholder content in screenshots

Play Store specific:
- [ ] Content rating questionnaire completed
- [ ] Privacy policy URL added
- [ ] Feature graphic uploaded (1024x500)
```
