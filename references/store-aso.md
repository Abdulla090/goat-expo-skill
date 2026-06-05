# Store Submission & ASO — App Store + Play Store Complete Guide

## THE SUBMISSION REALITY

Most apps get rejected on first submission. The most common reasons:
1. Missing privacy manifest (iOS 17.2+) — instant rejection
2. Broken UI on specific device sizes — rejection
3. Placeholder content in screenshots — rejection
4. Missing permission usage descriptions — rejection
5. Crash on reviewer's device — rejection

This file covers everything needed to pass on first submission
and rank well after approval.

---

## iOS — PRIVACY MANIFEST (MANDATORY FROM iOS 17.2)

Apple requires a `PrivacyInfo.xcprivacy` file declaring all:
- Privacy-sensitive APIs your app uses
- Third-party SDKs that use those APIs
- Reasons for each API access

**Without this file: your app will be rejected.**

### Create the privacy manifest

```bash
# In your Expo project
mkdir -p ios/YourApp
touch ios/YourApp/PrivacyInfo.xcprivacy
```

```xml
<!-- ios/YourApp/PrivacyInfo.xcprivacy -->
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN"
  "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
  <key>NSPrivacyAccessedAPITypes</key>
  <array>

    <!-- UserDefaults — required if using MMKV or AsyncStorage -->
    <dict>
      <key>NSPrivacyAccessedAPIType</key>
      <string>NSPrivacyAccessedAPICategoryUserDefaults</string>
      <key>NSPrivacyAccessedAPITypeReasons</key>
      <array>
        <string>CA92.1</string>  <!-- Read/write user preferences -->
      </array>
    </dict>

    <!-- File timestamp — required by many SDK internals -->
    <dict>
      <key>NSPrivacyAccessedAPIType</key>
      <string>NSPrivacyAccessedAPICategoryFileTimestamp</string>
      <key>NSPrivacyAccessedAPITypeReasons</key>
      <array>
        <string>C617.1</string>  <!-- Access to own app files -->
      </array>
    </dict>

    <!-- System boot time — required by some analytics SDKs -->
    <dict>
      <key>NSPrivacyAccessedAPIType</key>
      <string>NSPrivacyAccessedAPICategorySystemBootTime</string>
      <key>NSPrivacyAccessedAPITypeReasons</key>
      <array>
        <string>35F9.1</string>  <!-- Calculate time since event -->
      </array>
    </dict>

    <!-- Disk space — required by Sentry and crash reporters -->
    <dict>
      <key>NSPrivacyAccessedAPIType</key>
      <string>NSPrivacyAccessedAPICategoryDiskSpace</string>
      <key>NSPrivacyAccessedAPITypeReasons</key>
      <array>
        <string>E174.1</string>  <!-- Check space before writing large files -->
      </array>
    </dict>

  </array>

  <!-- Declare data collection (shown in App Store privacy label) -->
  <key>NSPrivacyCollectedDataTypes</key>
  <array>
    <!-- Add entries for data you actually collect -->
    <!-- Example: if you collect email addresses -->
    <dict>
      <key>NSPrivacyCollectedDataType</key>
      <string>NSPrivacyCollectedDataTypeEmailAddress</string>
      <key>NSPrivacyCollectedDataTypeLinked</key>
      <true/>
      <key>NSPrivacyCollectedDataTypeTracking</key>
      <false/>
      <key>NSPrivacyCollectedDataTypePurposes</key>
      <array>
        <string>NSPrivacyCollectedDataTypePurposeAppFunctionality</string>
      </array>
    </dict>
  </array>

  <!-- Tracking — set to true only if using ad tracking -->
  <key>NSPrivacyTracking</key>
  <false/>

</dict>
</plist>
```

Wire it in `app.json`:
```json
{
  "expo": {
    "plugins": [
      ["expo-build-properties", {
        "ios": {
          "privacyManifestPath": "./ios/YourApp/PrivacyInfo.xcprivacy"
        }
      }]
    ]
  }
}
```

### Common SDK privacy manifest requirements

| SDK | APIs it uses | Reason codes needed |
|---|---|---|
| Sentry | File timestamp, disk space, system boot time | C617.1, E174.1, 35F9.1 |
| MMKV | UserDefaults | CA92.1 |
| AsyncStorage | UserDefaults | CA92.1 |
| PostHog | System boot time | 35F9.1 |
| Reanimated | File timestamp | C617.1 |

Check each SDK's docs for their required entries — they often publish them.

---

## iOS — PERMISSION USAGE DESCRIPTIONS

Every permission your app requests needs a human-readable description in `app.json`.
Missing = rejection.

```json
{
  "expo": {
    "ios": {
      "infoPlist": {
        "NSCameraUsageDescription": "We need camera access to let you scan QR codes and take profile photos.",
        "NSPhotoLibraryUsageDescription": "We need photo library access to let you choose a profile picture.",
        "NSPhotoLibraryAddUsageDescription": "We need permission to save photos to your library.",
        "NSMicrophoneUsageDescription": "We need microphone access to record voice messages.",
        "NSLocationWhenInUseUsageDescription": "We use your location to show nearby results.",
        "NSLocationAlwaysAndWhenInUseUsageDescription": "We use your location in the background to send location-based reminders.",
        "NSContactsUsageDescription": "We access your contacts to help you invite friends.",
        "NSFaceIDUsageDescription": "We use Face ID to securely authenticate you.",
        "NSSpeechRecognitionUsageDescription": "We use speech recognition to transcribe your voice notes.",
        "NSUserNotificationsUsageDescription": "We send notifications to keep you updated on important activity."
      }
    }
  }
}
```

**Rules for descriptions:**
- Be specific about WHY, not just WHAT
- "to let you..." performs better with reviewers than passive descriptions
- Don't be vague — "for app functionality" gets rejected
- Must match what you actually do — reviewers test it

---

## iOS APP STORE — SUBMISSION PROCESS

### Step 1 — App Store Connect setup

1. Create app at appstoreconnect.apple.com
2. Set bundle ID (must match `expo.ios.bundleIdentifier` in app.json)
3. Set primary language, category, age rating
4. Fill App Privacy questionnaire completely

### Step 2 — Screenshots (most important ASO factor)

Required sizes:
```
iPhone 6.9" (required):  1320 × 2868 px  (iPhone 16 Pro Max)
iPhone 6.5" (required):  1284 × 2778 px  (iPhone 13 Pro Max / 14 Plus)
iPad 13"    (if iPad):   2064 × 2752 px
iPad 12.9"  (if iPad):   2048 × 2732 px
```

**Screenshot strategy that converts:**
```
Screenshot 1: The hero moment — your app's most impressive feature
Screenshot 2: Core use case — what users do 80% of the time
Screenshot 3: Social proof or key differentiator
Screenshot 4: Secondary feature
Screenshot 5: Onboarding or trust moment
```

Add caption text overlays on screenshots — apps with text outperform without by 35%.
Use device frames (optional but increases conversion).

### Step 3 — Metadata

```
App Name:     30 characters max — include primary keyword
Subtitle:     30 characters max — include secondary keyword
Description:  4000 characters — first 3 lines are above the fold (most read)
Keywords:     100 characters — comma-separated, no spaces, no duplicate words from name
```

### Step 4 — Submit build

```bash
# Build for App Store
eas build --platform ios --profile production

# Submit to App Store Connect
eas submit --platform ios

# Or manually: download .ipa from EAS → upload via Transporter app
```

### Step 5 — App Review

Average review time: 24–48 hours (first submission: up to 7 days).
If rejected: read rejection reason carefully, fix specifically, respond via Resolution Center.

---

## iOS — COMMON REJECTION REASONS & FIXES

| Rejection | Fix |
|---|---|
| 2.1 — App functionality broken | Test on real device matching reviewer's iOS version |
| 2.3.3 — Placeholder content in screenshots | Use real app UI, not mockups with Lorem Ipsum |
| 4.0 — Copycat / clone | Differentiate clearly in description, add unique features |
| 5.1.1 — Privacy policy missing | Add privacy policy URL in App Store Connect |
| 5.1.2 — Data collection not declared | Fill privacy manifest + App Store privacy label accurately |
| 2.5.4 — Background location | Only use if genuinely needed, add full justification |
| 4.8 — Sign in with Apple missing | If you offer Google/FB login, you must also offer Sign in with Apple |
| 3.1.1 — IAP bypass | Any purchase must go through Apple IAP — no external payment links |

### The Sign in with Apple rule

```bash
npx expo install expo-apple-authentication
```

```tsx
import * as AppleAuthentication from 'expo-apple-authentication';

// Required if your app has ANY third-party sign-in (Google, Facebook, etc.)
{Platform.OS === 'ios' && (
  <AppleAuthentication.AppleAuthenticationButton
    buttonType={AppleAuthentication.AppleAuthenticationButtonType.SIGN_IN}
    buttonStyle={AppleAuthentication.AppleAuthenticationButtonStyle.BLACK}
    cornerRadius={12}
    style={{ width: '100%', height: 48 }}
    onPress={async () => {
      const credential = await AppleAuthentication.signInAsync({
        requestedScopes: [
          AppleAuthentication.AppleAuthenticationScope.FULL_NAME,
          AppleAuthentication.AppleAuthenticationScope.EMAIL,
        ],
      });
      // credential.identityToken → send to your backend
    }}
  />
)}
```

---

## PLAY STORE — SUBMISSION PROCESS

### Step 1 — Play Console setup

1. Create app at play.google.com/console
2. Set package name (must match `expo.android.package` — **cannot change after publish**)
3. Complete Store listing (title, description, screenshots, feature graphic)
4. Complete Content rating questionnaire
5. Add privacy policy URL
6. Set app category and tags

### Step 2 — Screenshots

```
Phone (required):      16:9 or 9:16, min 320dp, max 3840px
7" tablet (optional):  landscape recommended
10" tablet (optional): landscape recommended
Feature graphic:       1024 × 500 px (shown in search results — very important)
```

Feature graphic is critical — it's the banner shown in search and featured placements.
Don't skip it. Use it as a mini billboard.

### Step 3 — Data safety form

Equivalent to iOS privacy manifest. Required.

```
Questions to answer:
1. Does your app collect any data? → Yes/No
2. For each data type collected:
   - Is it shared with third parties?
   - Is it used for tracking?
   - Is it required or optional?
   - Purpose (App functionality / Analytics / Developer communications / etc.)
```

Map your data collection to this form accurately. Mismatch = rejection.

### Step 4 — AAB upload + submit

```bash
# Build AAB
eas build --platform android --profile production

# Submit to Play Store (internal track first)
eas submit --platform android --track internal

# After testing: promote to production in Play Console
```

---

## ASO — APP STORE OPTIMIZATION

ASO = the difference between 100 downloads/day and 1000 downloads/day.
Most developers skip this. Don't.

### Keyword research

```
Tools:
- AppFollow.io — keyword volume + difficulty
- Sensor Tower (paid) — competitor keywords
- AppTweak (paid) — keyword rankings
- Free: search your competitors in App Store → their keywords are visible
```

**Keyword strategy:**
```
High volume + low competition  → primary keywords (in name + subtitle + keywords field)
High volume + high competition → secondary keywords (in description)
Long-tail (3+ words)          → put in description — less competition, more targeted
```

### iOS keyword field rules

```
- 100 characters total
- Comma-separated, no spaces after commas
- No duplicate words from your app name or subtitle
- No competitor names (violation)
- Include common misspellings if high-volume
- Localize for each market separately

Example (productivity app):
"task,planner,todo,reminder,calendar,schedule,organize,productivity,habit,focus"
```

### App name impact on rankings

```
iOS App Store: app name is the #1 ranking factor for keywords
Format: "Primary Name: Secondary Keyword Phrase"
Example: "Zana: AI Resume Builder" — ranks for both "Zana" and "AI Resume Builder"

Play Store: title (30 chars) + short description (80 chars) both indexed
Use primary keywords in both
```

### Description that converts

```
Structure:
Line 1–3: Hook — what does this app do in plain language?
Line 4–8: Top 3 features with emoji bullets
Line 9–15: Social proof / numbers ("Used by 50,000 professionals")
Line 16–20: Secondary features
Line 21+: Long-tail keywords in natural sentences

Example first 3 lines:
"Build an ATS-ready resume in minutes.
Zana analyzes job descriptions and tailors your resume automatically.
No formatting headaches. No guessing what recruiters want."
```

### Ratings & reviews strategy

```tsx
// Request review at the right moment — after user succeeds, not randomly
import * as StoreReview from 'expo-store-review';

async function requestReviewAfterSuccess() {
  // Only after a positive moment: finished a task, hit a milestone, etc.
  const isAvailable = await StoreReview.isAvailableAsync();
  if (isAvailable) {
    await StoreReview.requestReview();
  }
}

// Rules:
// - Max 3 times per year (iOS enforces this)
// - Only after clear user success (never on error screens)
// - After user has used the app enough to have an opinion (not first session)
```

```bash
npx expo install expo-store-review
```

### A/B test your store listing

Play Store: Go to Store listing → Create custom store listing → A/B test icons, screenshots, description.
App Store: Product Page Optimization (3 treatments, 90-day test).

Test one element at a time: icon → screenshots → description.
Run each test minimum 7 days before deciding.

---

## ASO CHECKLIST — BEFORE LAUNCH

```
App Name & Metadata:
- [ ] Primary keyword in app name
- [ ] Secondary keyword in subtitle (iOS) / short description (Android)
- [ ] 100-character keyword field fully used (iOS)
- [ ] Description: hook in first 3 lines, keywords throughout, benefits not features
- [ ] Privacy policy URL added (both stores)

Screenshots:
- [ ] All required sizes produced (iOS: 6.9" + 6.5", Android: phone + feature graphic)
- [ ] Screenshot 1 shows the most impressive feature
- [ ] Text overlays on screenshots
- [ ] No placeholder/Lorem Ipsum content
- [ ] Dark and light mode versions (optional but recommended)
- [ ] Screenshots match actual app UI exactly

Ratings:
- [ ] Review prompt wired to a positive user moment
- [ ] Sentry wired to monitor crash rate (bad reviews follow crashes)

Technical (iOS):
- [ ] Privacy manifest complete
- [ ] All permission descriptions filled
- [ ] Sign in with Apple if you have third-party login
- [ ] App Privacy questionnaire complete in App Store Connect

Technical (Android):
- [ ] Package name final (never changes)
- [ ] Data safety form complete
- [ ] Content rating questionnaire complete
- [ ] Feature graphic 1024×500 uploaded
- [ ] Internal track tested before production submission
```
