# Security — Token Storage, Network, Hardening (June 2026)

## SDK 56 NETWORK NOTE

Default `globalThis.fetch` is **`expo/fetch`**. Certificate pinning and custom agents may need explicit configuration — test TLS behavior after SDK upgrades.

---

## THE SECURITY HIERARCHY

Fix in this order. Most apps fail at layer 1-3.

```
1. Token storage          → never MMKV for auth tokens — use SecureStore (encrypted keychain)
2. Network security       → HTTPS everywhere, certificate pinning for sensitive apps
3. Sensitive screen guard → prevent screenshots on payment/auth screens
4. Deep link validation   → never trust incoming URL parameters blindly
5. Input sanitization     → never render raw HTML or evaluate user input
6. Jailbreak/root detect  → optional, for fintech/health apps
7. Code obfuscation       → Hermes bytecode + Proguard (Android)
```

---

## TOKEN STORAGE — THE #1 MISTAKE

**MMKV is NOT encrypted by default.** Tokens stored in MMKV can be extracted
from a rooted/jailbroken device or via backup extraction.

| Data type | Storage | Why |
|---|---|---|
| Auth token / refresh token | `expo-secure-store` | Encrypted, keychain-backed |
| Session ID | `expo-secure-store` | Encrypted |
| User preferences, theme, settings | MMKV | Fine — not sensitive |
| Cached API responses | MMKV | Fine — not sensitive |
| Payment tokens, PII | `expo-secure-store` | Encrypted |

```bash
npx expo install expo-secure-store
```

```tsx
import * as SecureStore from 'expo-secure-store';

// Store auth token — encrypted in iOS Keychain / Android Keystore
await SecureStore.setItemAsync('auth_token', token, {
  keychainAccessible: SecureStore.WHEN_UNLOCKED,  // iOS: only accessible when device unlocked
});

const token = await SecureStore.getItemAsync('auth_token');
await SecureStore.deleteItemAsync('auth_token');

// For high-security apps (finance, health): require biometrics to access
await SecureStore.setItemAsync('auth_token', token, {
  requireAuthentication: true,  // Requires Face ID / fingerprint each time
});
```

**Supabase + SecureStore setup:**
```tsx
// lib/supabase.ts
import { createClient } from '@supabase/supabase-js';
import * as SecureStore from 'expo-secure-store';

const SecureStoreAdapter = {
  getItem: (key: string) => SecureStore.getItemAsync(key),
  setItem: (key: string, value: string) => SecureStore.setItemAsync(key, value),
  removeItem: (key: string) => SecureStore.deleteItemAsync(key),
};

export const supabase = createClient(url, anonKey, {
  auth: {
    storage: SecureStoreAdapter,  // tokens go to Keychain, not AsyncStorage
    autoRefreshToken: true,
    persistSession: true,
    detectSessionInUrl: false,
  },
});
```

---

## NETWORK SECURITY

### HTTPS everywhere
Expo enforces HTTPS in production builds. If your API is HTTP-only,
add an exception (but fix the API first):

```json
// app.json — only as temporary measure
{
  "expo": {
    "ios": {
      "infoPlist": {
        "NSAppTransportSecurity": {
          "NSAllowsArbitraryLoads": false,
          "NSExceptionDomains": {
            "your-dev-api.com": {
              "NSExceptionAllowsInsecureHTTPLoads": true
            }
          }
        }
      }
    }
  }
}
```

### Certificate Pinning (for fintech, health, sensitive data apps)
Prevents man-in-the-middle attacks even on compromised networks.

```bash
npm install react-native-ssl-pinning
```

```tsx
import { fetch } from 'react-native-ssl-pinning';

// Only proceed if server certificate matches your pinned cert
const response = await fetch('https://api.myapp.com/users', {
  method: 'GET',
  sslPinning: {
    certs: ['cert_sha256_hash_here'],  // SHA256 hash of your server's certificate
  },
  headers: { Authorization: `Bearer ${token}` },
});
```

Get your cert hash:
```bash
openssl s_client -connect api.myapp.com:443 | openssl x509 -pubkey -noout | openssl pkey -pubin -outform der | openssl dgst -sha256 -binary | base64
```

**When to use:** apps handling payment info, medical data, private messages.
**When to skip:** general consumer apps where the attack surface is lower.

---

## SENSITIVE SCREEN PROTECTION

Prevent screenshot and screen recording on sensitive screens (payment, auth, PII).

```bash
npx expo install expo-screen-capture
```

```tsx
import * as ScreenCapture from 'expo-screen-capture';
import { useFocusEffect } from 'expo-router';

// Prevent screenshots on this screen only
function PaymentScreen() {
  useFocusEffect(() => {
    ScreenCapture.preventScreenCaptureAsync();
    return () => {
      ScreenCapture.allowScreenCaptureAsync();  // restore when leaving screen
    };
  });

  return <PaymentForm />;
}
```

Apply to: payment screens, password/PIN entry, document viewers with sensitive data.

---

## DEEP LINK VALIDATION

Deep links are external inputs — treat them like user input: never trust, always validate.

```tsx
// app/_layout.tsx — validate all incoming deep links
import { useURL } from 'expo-linking';

function DeepLinkHandler() {
  const url = useURL();

  useEffect(() => {
    if (!url) return;
    handleDeepLink(url);
  }, [url]);
}

function handleDeepLink(url: string) {
  // 1. Parse the URL
  const parsed = Linking.parse(url);

  // 2. Validate scheme — reject unknown schemes
  const allowedSchemes = ['myapp', 'https'];
  if (!allowedSchemes.includes(parsed.scheme ?? '')) {
    console.warn('Rejected deep link with unknown scheme:', parsed.scheme);
    return;
  }

  // 3. Validate path — reject unknown paths
  const allowedPaths = ['/post/', '/profile/', '/reset-password'];
  const isAllowed = allowedPaths.some(p => parsed.path?.startsWith(p));
  if (!isAllowed) {
    console.warn('Rejected deep link with unknown path:', parsed.path);
    return;
  }

  // 4. Validate params — sanitize before use
  const postId = parsed.queryParams?.id;
  if (postId && !/^[a-zA-Z0-9_-]+$/.test(String(postId))) {
    console.warn('Rejected deep link with suspicious param:', postId);
    return;
  }

  // 5. Safe to navigate
  router.push(parsed.path!);
}
```

**Never do:**
```tsx
// ❌ Blindly navigate to any deep link path
router.push(url.path);

// ❌ Eval or render HTML from deep link params
<WebView source={{ html: params.content }} />
```

---

## INPUT SECURITY

### Never render raw HTML from user input
```tsx
// ❌ XSS vector if content comes from user or API
<WebView source={{ html: userContent }} />

// ✅ Use a safe markdown renderer
import Markdown from 'react-native-markdown-display';
<Markdown>{userContent}</Markdown>

// ✅ Or strip HTML and render as text
import { decode } from 'html-entities';
<Text>{decode(userContent.replace(/<[^>]*>/g, ''))}</Text>
```

### Sanitize before sending to API
```tsx
// Zod does this automatically — schema rejects invalid shapes
const schema = z.object({
  username: z.string()
    .min(3).max(30)
    .regex(/^[a-zA-Z0-9_]+$/, 'Only letters, numbers, underscores'),
  bio: z.string().max(160).trim(),
});

// Additional: trim all string inputs
const sanitizedData = {
  username: data.username.trim().toLowerCase(),
  bio: data.bio.trim(),
};
```

---

## SENSITIVE DATA IN MEMORY

### Clear sensitive data when app backgrounds
```tsx
import { AppState } from 'react-native';

// Clear clipboard if it contains sensitive data (e.g. after copy OTP)
useEffect(() => {
  const subscription = AppState.addEventListener('change', (state) => {
    if (state === 'background' || state === 'inactive') {
      Clipboard.setStringAsync('');  // clear clipboard
      // Also clear any in-memory sensitive state
    }
  });
  return () => subscription.remove();
}, []);
```

### Don't log sensitive data
```tsx
// ❌ Token visible in logs
console.log('Auth response:', response);

// ✅ Log only what's safe
console.log('Auth success, user ID:', response.user.id);

// In production: drop ALL console.logs via metro config
// metro.config.js
config.transformer.minifierConfig = {
  compress: { drop_console: true }
};
```

---

## JAILBREAK / ROOT DETECTION

For fintech, health, enterprise apps. Optional for general consumer apps.

```bash
npm install @doko/expo-jailbreak-detector
```

```tsx
import { isJailbroken } from '@doko/expo-jailbreak-detector';

async function checkDeviceSecurity() {
  const jailbroken = await isJailbroken();

  if (jailbroken) {
    // Options:
    // 1. Block entirely (fintech requirement)
    Alert.alert('Security', 'This app cannot run on jailbroken/rooted devices.');
    // 2. Warn and continue (less strict)
    // 3. Limit features (no biometric auth, no offline mode)
  }
}
```

---

## BIOMETRIC AUTHENTICATION

```bash
npx expo install expo-local-authentication
```

```tsx
import * as LocalAuthentication from 'expo-local-authentication';

async function authenticateWithBiometrics(): Promise<boolean> {
  // Check if biometrics are available
  const hasHardware = await LocalAuthentication.hasHardwareAsync();
  const isEnrolled = await LocalAuthentication.isEnrolledAsync();

  if (!hasHardware || !isEnrolled) {
    // Fall back to PIN/password
    return authenticateWithPIN();
  }

  const result = await LocalAuthentication.authenticateAsync({
    promptMessage: 'Authenticate to continue',
    fallbackLabel: 'Use passcode',
    cancelLabel: 'Cancel',
    disableDeviceFallback: false,
  });

  return result.success;
}

// Re-authenticate on sensitive actions (payment, settings change)
async function handleDeleteAccount() {
  const authenticated = await authenticateWithBiometrics();
  if (!authenticated) return;

  // Proceed with destructive action
  await api.deleteAccount();
}
```

---

## APP HARDENING — CODE OBFUSCATION

### Hermes Bytecode (automatic in production)
Expo production builds compile JS to Hermes bytecode — not readable source.
Raw JS strings are not visible in the binary.
Enable by default via EAS production profile. Nothing to configure.

### Android Proguard (shrinks + obfuscates Java/Kotlin)
```json
// app.json
{
  "expo": {
    "android": {
      "enableProguardInReleaseBuilds": true,
      "enableR8FullMode": true
    }
  }
}
```

R8 full mode = stronger optimization + obfuscation. Enable for production.

### What not to put in the JS bundle
- Private API keys (use EAS Secrets for build-time, server-side for runtime)
- Encryption keys (use SecureStore + server-side key management)
- Admin credentials (never in client code)

```
EXPO_PUBLIC_* = in JS bundle = readable (even with Hermes)
EAS Secrets   = build-time only = NOT in JS bundle
```

---

## SECURITY CHECKLIST (pre-submission)

```
Auth & Storage:
- [ ] Auth tokens in expo-secure-store, NOT MMKV or AsyncStorage
- [ ] Supabase client configured with SecureStore adapter
- [ ] No sensitive data in MMKV or AsyncStorage
- [ ] No API keys in EXPO_PUBLIC_* variables
- [ ] console.logs dropped in production build

Network:
- [ ] All API calls use HTTPS
- [ ] Certificate pinning added (if fintech/health/private messages)
- [ ] API keys not hardcoded in source

Data:
- [ ] Deep links validated before navigation
- [ ] User input sanitized with Zod before API submission
- [ ] No raw HTML rendering from user content
- [ ] Clipboard cleared after sensitive copy operations

Platform:
- [ ] Screenshot prevention on payment/auth screens
- [ ] Biometric auth on sensitive actions (optional but recommended)
- [ ] Android Proguard + R8 full mode enabled
- [ ] Jailbreak detection (required for fintech/health)
```
