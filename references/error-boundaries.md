# Error Boundaries & Crash Recovery (June 2026)

SDK 56 Sentry + fetch:
- Default **`expo/fetch`** → `traceFetch: true` when HTTP spans are missing
- **`EXPO_PUBLIC_USE_RN_FETCH=1`** → set `traceFetch: false` (avoid duplicate breadcrumbs/spans)

See `references/production-edge-cases.md`.

## THE RULE

Every React Native app will crash. The question is whether the user
sees a white screen and force-quits, or sees a recovery screen and continues.

Error boundaries catch JS errors. Sentry logs them. Recovery screens
give users a path forward. All three are required.

---

## REACT ERROR BOUNDARY

React Error Boundaries catch errors during render, in lifecycle methods,
and in constructors of child components. They do NOT catch:
- Async errors (useEffect, event handlers) — handle with try/catch
- Native crashes — handle with Sentry native crash reporting

```tsx
// components/ErrorBoundary.tsx
import React, { Component, ErrorInfo } from 'react';
import * as Sentry from '@sentry/react-native';

interface Props {
  children: React.ReactNode;
  fallback?: React.ComponentType<{ error: Error; resetError: () => void }>;
  onError?: (error: Error, errorInfo: ErrorInfo) => void;
}

interface State {
  hasError: boolean;
  error: Error | null;
}

export class ErrorBoundary extends Component<Props, State> {
  constructor(props: Props) {
    super(props);
    this.state = { hasError: false, error: null };
  }

  static getDerivedStateFromError(error: Error): State {
    return { hasError: true, error };
  }

  componentDidCatch(error: Error, errorInfo: ErrorInfo) {
    // Report to Sentry with component stack
    Sentry.captureException(error, {
      contexts: {
        react: {
          componentStack: errorInfo.componentStack,
        },
      },
    });

    // Call optional onError prop
    this.props.onError?.(error, errorInfo);
  }

  resetError = () => {
    this.setState({ hasError: false, error: null });
  };

  render() {
    if (this.state.hasError && this.state.error) {
      const FallbackComponent = this.props.fallback ?? DefaultErrorFallback;
      return (
        <FallbackComponent
          error={this.state.error}
          resetError={this.resetError}
        />
      );
    }

    return this.props.children;
  }
}
```

---

## DEFAULT ERROR FALLBACK — PRODUCTION QUALITY

```tsx
// components/DefaultErrorFallback.tsx
import { View, Text, Pressable, ScrollView } from 'react-native';
import { useSafeAreaInsets } from 'react-native-safe-area-context';
import * as Updates from 'expo-updates';

interface Props {
  error: Error;
  resetError: () => void;
}

export function DefaultErrorFallback({ error, resetError }: Props) {
  const insets = useSafeAreaInsets();
  const [showDetails, setShowDetails] = useState(false);

  async function handleReloadApp() {
    try {
      await Updates.reloadAsync();
    } catch {
      // Fallback if OTA reload fails — just reset the boundary
      resetError();
    }
  }

  return (
    <View
      style={{
        flex: 1,
        paddingTop: insets.top,
        paddingBottom: insets.bottom,
        paddingHorizontal: 24,
        backgroundColor: '#fff',
        alignItems: 'center',
        justifyContent: 'center',
      }}
    >
      {/* Icon */}
      <View style={{
        width: 80,
        height: 80,
        borderRadius: 20,
        backgroundColor: '#fef2f2',
        alignItems: 'center',
        justifyContent: 'center',
        marginBottom: 24,
      }}>
        <Text style={{ fontSize: 36 }}>⚠️</Text>
      </View>

      {/* Message */}
      <Text style={{ fontSize: 22, fontWeight: '700', textAlign: 'center', marginBottom: 12 }}>
        Something went wrong
      </Text>
      <Text style={{ fontSize: 16, color: '#64748b', textAlign: 'center', lineHeight: 24, marginBottom: 32 }}>
        The app ran into an unexpected error. This has been reported automatically.
      </Text>

      {/* Primary action */}
      <Pressable
        onPress={handleReloadApp}
        style={{
          backgroundColor: '#0ea5e9',
          borderRadius: 14,
          paddingVertical: 16,
          paddingHorizontal: 32,
          width: '100%',
          alignItems: 'center',
          marginBottom: 12,
        }}
      >
        <Text style={{ color: '#fff', fontSize: 16, fontWeight: '600' }}>
          Restart App
        </Text>
      </Pressable>

      {/* Secondary action — try to recover without restart */}
      <Pressable
        onPress={resetError}
        style={{
          borderRadius: 14,
          paddingVertical: 16,
          paddingHorizontal: 32,
          width: '100%',
          alignItems: 'center',
        }}
      >
        <Text style={{ color: '#64748b', fontSize: 16 }}>
          Try to continue
        </Text>
      </Pressable>

      {/* Error details (dev only) */}
      {__DEV__ && (
        <Pressable onPress={() => setShowDetails(!showDetails)} style={{ marginTop: 24 }}>
          <Text style={{ color: '#94a3b8', fontSize: 13 }}>
            {showDetails ? 'Hide' : 'Show'} error details
          </Text>
        </Pressable>
      )}

      {__DEV__ && showDetails && (
        <ScrollView style={{ marginTop: 12, maxHeight: 200 }}>
          <Text style={{ fontSize: 11, color: '#ef4444', fontFamily: 'monospace' }}>
            {error.name}: {error.message}{'\n\n'}{error.stack}
          </Text>
        </ScrollView>
      )}
    </View>
  );
}
```

---

## WIRING ERROR BOUNDARIES

### Level 1 — Root boundary (catches everything)

```tsx
// app/_layout.tsx
import * as Sentry from '@sentry/react-native';
import { ErrorBoundary } from '../components/ErrorBoundary';

export default Sentry.wrap(function RootLayout() {
  return (
    <ErrorBoundary>
      <SafeAreaProvider>
        <QueryClientProvider client={queryClient}>
          <GestureHandlerRootView style={{ flex: 1 }}>
            <Stack />
          </GestureHandlerRootView>
        </QueryClientProvider>
      </SafeAreaProvider>
    </ErrorBoundary>
  );
});
```

### Level 2 — Screen boundary (isolate screen crashes)

```tsx
// Don't let one screen crash the whole app
// app/(tabs)/feed.tsx
export default function FeedScreen() {
  return (
    <ErrorBoundary
      fallback={({ resetError }) => (
        <View className="flex-1 items-center justify-center">
          <Text>Feed couldn't load</Text>
          <Pressable onPress={resetError}>
            <Text className="text-blue-500 mt-4">Retry</Text>
          </Pressable>
        </View>
      )}
    >
      <FeedContent />
    </ErrorBoundary>
  );
}
```

### Level 3 — Component boundary (isolate widget crashes)

```tsx
// A broken widget shouldn't break the screen
<ErrorBoundary
  fallback={() => <View className="h-40 bg-gray-100 rounded-xl" />}  // silent fallback
>
  <ComplexWidget />
</ErrorBoundary>
```

---

## ASYNC ERROR HANDLING — NOT CAUGHT BY BOUNDARIES

Error Boundaries only catch render errors. Async errors need explicit handling:

```tsx
// Global unhandled promise rejection handler
// Add to app/_layout.tsx, before render
if (typeof global !== 'undefined') {
  const originalHandler = global.onunhandledrejection;
  global.onunhandledrejection = (event: PromiseRejectionEvent) => {
    Sentry.captureException(event.reason);
    originalHandler?.(event);
  };
}

// Per-useEffect error handling
useEffect(() => {
  async function loadData() {
    try {
      const data = await fetchCriticalData();
      setData(data);
    } catch (error) {
      Sentry.captureException(error);
      setError(error);
    }
  }
  loadData();
}, []);

// Event handler error handling
async function handleSubmit() {
  try {
    await submitForm(data);
  } catch (error) {
    Sentry.captureException(error);
    showErrorToast('Failed to submit. Please try again.');
  }
}
```

---

## SENTRY INTEGRATION — FULL SETUP

```tsx
// app/_layout.tsx
import * as Sentry from '@sentry/react-native';

Sentry.init({
  dsn: process.env.EXPO_PUBLIC_SENTRY_DSN,

  // Environment tagging
  environment: __DEV__ ? 'development' : 'production',
  release: `${Application.applicationId}@${Application.nativeApplicationVersion}`,

  // Performance monitoring
  tracesSampleRate: __DEV__ ? 1.0 : 0.1,  // 100% in dev, 10% in prod
  profilesSampleRate: __DEV__ ? 1.0 : 0.1,

  // Breadcrumbs — track user actions leading to crash
  attachStacktrace: true,
  maxBreadcrumbs: 50,

  // Don't send in dev (noisy)
  enabled: !__DEV__,

  // Ignore known non-critical errors
  ignoreErrors: [
    'Network request failed',      // offline — not a bug
    'Load failed',                 // image load failed — not a crash
  ],

  // Add user context when authenticated
  beforeSend(event) {
    // Add user info to crash reports
    return event;
  },
});

// Set user context after login (for crash reports to show affected user)
export function identifySentryUser(userId: string, email: string) {
  Sentry.setUser({ id: userId, email });
}

// Clear on logout
export function clearSentryUser() {
  Sentry.setUser(null);
}

// Manual error capture (in catch blocks)
export function captureError(error: unknown, context?: Record<string, unknown>) {
  Sentry.captureException(error, { extra: context });
}
```

### Sentry breadcrumbs — track user path to crash

```tsx
// Add breadcrumbs at key moments
import * as Sentry from '@sentry/react-native';

// Navigation
router.push('/checkout');
Sentry.addBreadcrumb({
  category: 'navigation',
  message: 'Navigated to checkout',
  level: 'info',
});

// User actions
function handleAddToCart(productId: string) {
  Sentry.addBreadcrumb({
    category: 'user_action',
    message: 'Added to cart',
    data: { productId },
    level: 'info',
  });
  addToCart(productId);
}
```

---

## NETWORK ERROR GLOBAL HANDLING

```tsx
// lib/api.ts — catch and log all API errors globally
api.interceptors.response.use(
  response => response,
  error => {
    const status = error.response?.status;

    // Log unexpected errors to Sentry (not 4xx client errors)
    if (!status || status >= 500) {
      Sentry.captureException(error, {
        extra: {
          url: error.config?.url,
          method: error.config?.method,
          status,
        },
      });
    }

    // 401 → auto-logout
    if (status === 401) {
      useAuthStore.getState().logout();
      router.replace('/login');
    }

    return Promise.reject(error);
  }
);
```

---

## CRASH RECOVERY CHECKLIST

```
Error Boundaries:
- [ ] Root ErrorBoundary wraps entire app
- [ ] Sentry.wrap() wraps root layout
- [ ] Screen-level boundaries on complex screens
- [ ] Component-level boundaries on complex widgets

Sentry:
- [ ] Sentry.init() called before render
- [ ] DSN in environment variable
- [ ] tracesSampleRate set (not 1.0 in production)
- [ ] User identified after login / cleared on logout
- [ ] Breadcrumbs added at key navigation points

Async Errors:
- [ ] try/catch in all useEffect async functions
- [ ] try/catch in all event handlers that call API
- [ ] Global unhandled rejection handler set

Recovery UI:
- [ ] DefaultErrorFallback has "Restart App" + "Try to continue" actions
- [ ] Error fallback never shows raw error message to users
- [ ] Error details shown only in __DEV__ mode
- [ ] Recovery screen matches app design (not generic white screen)
```
