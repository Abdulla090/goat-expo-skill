# Universal Links & Deep Links — iOS + Android Complete Setup

## THE DIFFERENCE

```
Deep Link:        myapp://post/123          → opens app if installed, fails if not
Universal Link:   https://myapp.com/post/123 → opens app if installed, website if not
App Link:         https://myapp.com/post/123 → Android equivalent of Universal Link
```

For production apps: implement Universal Links (iOS) + App Links (Android).
They're more reliable, work in more contexts (email, SMS, web), and are required for:
- Password reset flows
- Email verification
- Social auth callbacks (Google, Facebook)
- Shared content links

---

## EXPO ROUTER DEEP LINK CONFIG

```json
// app.json
{
  "expo": {
    "scheme": "myapp",
    "ios": {
      "associatedDomains": ["applinks:myapp.com"]
    },
    "android": {
      "intentFilters": [
        {
          "action": "VIEW",
          "autoVerify": true,
          "data": [
            {
              "scheme": "https",
              "host": "myapp.com",
              "pathPrefix": "/"
            }
          ],
          "category": ["BROWSABLE", "DEFAULT"]
        }
      ]
    }
  }
}
```

expo-router handles routing automatically based on your file structure:
- `https://myapp.com/post/123` → `app/post/[id].tsx` with `id = "123"`
- `https://myapp.com/profile/abdulla` → `app/profile/[username].tsx`
- `myapp://reset-password?token=abc` → `app/reset-password.tsx`

---

## iOS — UNIVERSAL LINKS SETUP

### Step 1 — Apple App Site Association (AASA) file

Host this file at: `https://myapp.com/.well-known/apple-app-site-association`
(no file extension, no redirect, must be served with `application/json` content-type)

```json
{
  "applinks": {
    "apps": [],
    "details": [
      {
        "appIDs": ["TEAMID.com.mycompany.myapp"],
        "components": [
          {
            "/": "/post/*",
            "comment": "Post pages"
          },
          {
            "/": "/profile/*",
            "comment": "Profile pages"
          },
          {
            "/": "/reset-password",
            "comment": "Password reset"
          },
          {
            "/": "/verify-email",
            "comment": "Email verification"
          }
        ]
      }
    ]
  },
  "webcredentials": {
    "apps": ["TEAMID.com.mycompany.myapp"]
  }
}
```

**Get your Team ID:** Apple Developer Portal → Membership → Team ID (10-character string)

### Step 2 — Verify AASA is accessible

```bash
# Test your AASA file
curl -I https://myapp.com/.well-known/apple-app-site-association
# Must return 200, Content-Type: application/json, no redirect

# Validate with Apple's tool
open https://app-site-association.cdn-apple.com/a/v1/myapp.com
```

### Step 3 — Configure in app.json (already done above)

```json
"ios": {
  "associatedDomains": ["applinks:myapp.com"]
}
```

This adds the Associated Domains entitlement to your app.
Requires EAS dev build — doesn't work in Expo Go.

---

## ANDROID — APP LINKS SETUP

### Step 1 — Digital Asset Links file

Host this file at: `https://myapp.com/.well-known/assetlinks.json`

```json
[
  {
    "relation": ["delegate_permission/common.handle_all_urls"],
    "target": {
      "namespace": "android_app",
      "package_name": "com.mycompany.myapp",
      "sha256_cert_fingerprints": [
        "YOUR_SHA256_FINGERPRINT_HERE"
      ]
    }
  }
]
```

**Get your SHA-256 fingerprint:**

```bash
# From EAS credentials
eas credentials --platform android
# Look for "SHA-256 certificate fingerprint"

# From keystore directly
keytool -list -v -keystore your.keystore | grep SHA256

# Format: AA:BB:CC:DD:EE:FF:... (colon-separated hex pairs)
```

### Step 2 — Verify App Links

```bash
# Test your assetlinks.json
curl https://myapp.com/.well-known/assetlinks.json

# Validate with Google's tool
open "https://digitalassetlinks.googleapis.com/v1/statements:list?source.web.site=https://myapp.com&relation=delegate_permission/common.handle_all_urls"

# Test on device (adb)
adb shell am start -a android.intent.action.VIEW \
  -c android.intent.category.BROWSABLE \
  -d "https://myapp.com/post/123"
```

---

## HANDLING INCOMING LINKS IN EXPO ROUTER

expo-router automatically routes incoming links to the correct screen.
For custom handling (auth callbacks, special params):

```tsx
// app/_layout.tsx
import { useURL } from 'expo-linking';
import { useEffect } from 'react';

export default function RootLayout() {
  const url = useURL();

  useEffect(() => {
    if (!url) return;
    handleIncomingLink(url);
  }, [url]);

  return <Stack />;
}

function handleIncomingLink(url: string) {
  const parsed = Linking.parse(url);

  // Password reset — extract token from URL
  if (parsed.path === '/reset-password') {
    const token = parsed.queryParams?.token as string;
    if (!token || !/^[a-zA-Z0-9_-]{20,}$/.test(token)) {
      console.warn('Invalid reset token in deep link');
      return;
    }
    router.push({ pathname: '/reset-password', params: { token } });
    return;
  }

  // Email verification
  if (parsed.path === '/verify-email') {
    const token = parsed.queryParams?.token as string;
    verifyEmail(token);
    return;
  }

  // OAuth callback (Google, Apple)
  if (parsed.path === '/auth/callback') {
    handleOAuthCallback(parsed.queryParams);
    return;
  }
}
```

---

## SOCIAL AUTH DEEP LINK SETUP

### Google Sign-In callback

```tsx
import * as WebBrowser from 'expo-web-browser';
import * as Google from 'expo-auth-session/providers/google';

WebBrowser.maybeCompleteAuthSession();  // Required — handles redirect back

export default function LoginScreen() {
  const [request, response, promptAsync] = Google.useIdTokenAuthRequest({
    clientId: process.env.EXPO_PUBLIC_GOOGLE_CLIENT_ID,
    iosClientId: process.env.EXPO_PUBLIC_GOOGLE_IOS_CLIENT_ID,
    androidClientId: process.env.EXPO_PUBLIC_GOOGLE_ANDROID_CLIENT_ID,
    redirectUri: makeRedirectUri({ scheme: 'myapp' }),
  });

  useEffect(() => {
    if (response?.type === 'success') {
      const { id_token } = response.params;
      // Send id_token to your backend
      handleGoogleAuth(id_token);
    }
  }, [response]);

  return (
    <Pressable onPress={() => promptAsync()} disabled={!request}>
      <Text>Sign in with Google</Text>
    </Pressable>
  );
}
```

---

## LINK SHARING — GENERATING LINKS

```tsx
import * as Linking from 'expo-linking';
import { Share } from 'react-native';

// Generate a universal link for sharing
function getPostLink(postId: string): string {
  // In production: use your web domain (universal link)
  return `https://myapp.com/post/${postId}`;
  // In dev: use expo scheme
  // return Linking.createURL(`/post/${postId}`);
}

async function sharePost(post: Post) {
  await Share.share({
    message: `Check out this post: ${post.title}`,
    url: getPostLink(post.id),
    title: post.title,
  });
}
```

---

## DEEP LINK TESTING

```bash
# iOS Simulator
xcrun simctl openurl booted "myapp://post/123"
xcrun simctl openurl booted "https://myapp.com/post/123"

# Android Emulator
adb shell am start -a android.intent.action.VIEW -d "myapp://post/123"
adb shell am start -a android.intent.action.VIEW -d "https://myapp.com/post/123"

# Test all critical paths:
# - Password reset: myapp://reset-password?token=testtoken
# - Email verify: myapp://verify-email?token=testtoken
# - Share link: https://myapp.com/post/test123
# - Profile link: https://myapp.com/profile/username
```

---

## DEEP LINK CHECKLIST

```
iOS Universal Links:
- [ ] AASA file hosted at /.well-known/apple-app-site-association
- [ ] AASA served as application/json with no redirect
- [ ] AASA validated with Apple's CDN tool
- [ ] associatedDomains in app.json
- [ ] Team ID correct in AASA (10-character alphanumeric)

Android App Links:
- [ ] assetlinks.json hosted at /.well-known/assetlinks.json
- [ ] SHA-256 fingerprint from EAS credentials (not dev keystore)
- [ ] autoVerify: true in intentFilters
- [ ] Validated with Google's tool

Code:
- [ ] useURL() listener in root layout
- [ ] All incoming paths validated before navigation
- [ ] Token params validated with regex before use
- [ ] WebBrowser.maybeCompleteAuthSession() called for OAuth

Testing:
- [ ] All link types tested on real device (not just simulator)
- [ ] Password reset flow end-to-end tested
- [ ] Social auth callback tested
- [ ] Fallback to website tested (when app not installed)
```
