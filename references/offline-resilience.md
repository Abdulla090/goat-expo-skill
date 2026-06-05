# Offline & Network Resilience (June 2026)

TanStack Query `networkMode: 'offlineFirst'` + MMKV cache persist + NetInfo `isInternetReachable`.

## THE RULE

In Iraq, Kurdistan, and most of the world outside major cities:
- Mobile connectivity drops constantly
- Users switch between WiFi and cellular mid-session
- Background data restrictions kill network calls
- Slow 3G is more common than fast 5G

An app that crashes or shows blank screens offline = uninstalled.
An app that works offline = trusted.

---

## NETINFO — DETECT CONNECTIVITY

```bash
npx expo install @react-native-community/netinfo
```

```tsx
import NetInfo from '@react-native-community/netinfo';

// One-time check
const state = await NetInfo.fetch();
console.log('Connected:', state.isConnected);
console.log('Type:', state.type);  // 'wifi' | 'cellular' | 'none' | 'unknown'
console.log('Is internet reachable:', state.isInternetReachable);

// Subscribe to changes
const unsubscribe = NetInfo.addEventListener(state => {
  if (!state.isConnected) {
    // User just went offline
    showOfflineBanner();
  } else {
    // User came back online
    hideOfflineBanner();
    syncPendingMutations();  // flush offline queue
  }
});

// Clean up
return () => unsubscribe();
```

### The isConnected vs isInternetReachable distinction

```tsx
// isConnected = has a network interface (WiFi connected, cellular has signal)
// isInternetReachable = can actually reach the internet

// Case: connected to WiFi router but router has no internet → isConnected: true, isInternetReachable: false
// Always check isInternetReachable for actual connectivity
const isOnline = state.isConnected && state.isInternetReachable !== false;
```

---

## GLOBAL OFFLINE BANNER

```tsx
// hooks/useNetworkStatus.ts
import { useState, useEffect } from 'react';
import NetInfo from '@react-native-community/netinfo';

export function useNetworkStatus() {
  const [isOnline, setIsOnline] = useState(true);

  useEffect(() => {
    const unsubscribe = NetInfo.addEventListener(state => {
      setIsOnline(state.isConnected === true && state.isInternetReachable !== false);
    });
    return unsubscribe;
  }, []);

  return isOnline;
}

// components/OfflineBanner.tsx
import { MotiView } from 'moti';
import { useNetworkStatus } from '../hooks/useNetworkStatus';

export function OfflineBanner() {
  const isOnline = useNetworkStatus();
  const insets = useSafeAreaInsets();

  return (
    <MotiView
      animate={{
        translateY: isOnline ? -60 : 0,
        opacity: isOnline ? 0 : 1,
      }}
      transition={{ type: 'spring', damping: 20 }}
      style={{
        position: 'absolute',
        top: insets.top,
        left: 0,
        right: 0,
        backgroundColor: '#ef4444',
        padding: 12,
        alignItems: 'center',
        zIndex: 9999,
      }}
    >
      <Text style={{ color: 'white', fontWeight: '600' }}>
        No internet connection
      </Text>
    </MotiView>
  );
}

// Add to root layout
export default function RootLayout() {
  return (
    <SafeAreaProvider>
      <OfflineBanner />
      <Stack />
    </SafeAreaProvider>
  );
}
```

---

## TANSTACK QUERY OFFLINE STRATEGY

TanStack Query handles offline gracefully with the right config:

```tsx
const queryClient = new QueryClient({
  defaultOptions: {
    queries: {
      // Show cached data immediately — even if stale
      staleTime: 1000 * 60 * 5,    // 5 minutes
      gcTime: 1000 * 60 * 60 * 24, // keep in cache 24 hours

      // Retry failed requests when back online
      retry: (failureCount, error) => {
        // Don't retry 4xx errors (client errors)
        if (error?.response?.status >= 400 && error?.response?.status < 500) {
          return false;
        }
        return failureCount < 3;
      },

      // Refetch when app comes back to foreground
      refetchOnWindowFocus: true,

      // Don't refetch when offline — wait for reconnect
      networkMode: 'offlineFirst',
    },
    mutations: {
      // Queue mutations when offline — retry when back online
      networkMode: 'offlineFirst',
      retry: 3,
    },
  },
});
```

### Persist TanStack Query cache to MMKV (survive app restarts)

```bash
npm install @tanstack/query-async-storage-persister @tanstack/react-query-persist-client
```

```tsx
import { PersistQueryClientProvider } from '@tanstack/react-query-persist-client';
import { createAsyncStoragePersister } from '@tanstack/query-async-storage-persister';
import { storage } from '../lib/storage';

// MMKV adapter for TanStack persister
const mmkvPersister = createAsyncStoragePersister({
  storage: {
    getItem: async (key) => storage.getString(key) ?? null,
    setItem: async (key, value) => storage.set(key, value),
    removeItem: async (key) => storage.delete(key),
  },
  throttleTime: 1000,  // write to storage max once per second
});

// Replace QueryClientProvider with PersistQueryClientProvider
export default function RootLayout() {
  return (
    <PersistQueryClientProvider
      client={queryClient}
      persistOptions={{
        persister: mmkvPersister,
        maxAge: 1000 * 60 * 60 * 24,  // cache valid for 24 hours
        buster: APP_VERSION,            // clear cache when app version changes
      }}
    >
      <App />
    </PersistQueryClientProvider>
  );
}
```

Now when user opens app offline:
- Cached data from last session renders immediately
- App is usable without internet
- Fresh data replaces cache when connectivity returns

---

## OFFLINE MUTATION QUEUE

For actions that must be performed even when offline (send message, save draft, sync data):

```tsx
// stores/offline-queue.store.ts
import { create } from 'zustand';
import { persist, createJSONStorage } from 'zustand/middleware';
import { storage } from '../lib/storage';

interface PendingMutation {
  id: string;
  type: string;
  payload: unknown;
  createdAt: number;
  retryCount: number;
}

interface OfflineQueueStore {
  queue: PendingMutation[];
  addToQueue: (mutation: Omit<PendingMutation, 'id' | 'createdAt' | 'retryCount'>) => void;
  removeFromQueue: (id: string) => void;
  incrementRetry: (id: string) => void;
}

const mmkvStorage = {
  getItem: (key: string) => storage.getString(key) ?? null,
  setItem: (key: string, value: string) => storage.set(key, value),
  removeItem: (key: string) => storage.delete(key),
};

export const useOfflineQueue = create<OfflineQueueStore>()(
  persist(
    (set) => ({
      queue: [],
      addToQueue: (mutation) => set((state) => ({
        queue: [...state.queue, {
          ...mutation,
          id: Date.now().toString(),
          createdAt: Date.now(),
          retryCount: 0,
        }],
      })),
      removeFromQueue: (id) => set((state) => ({
        queue: state.queue.filter(m => m.id !== id),
      })),
      incrementRetry: (id) => set((state) => ({
        queue: state.queue.map(m =>
          m.id === id ? { ...m, retryCount: m.retryCount + 1 } : m
        ),
      })),
    }),
    {
      name: 'offline-queue',
      storage: createJSONStorage(() => mmkvStorage),
    }
  )
);

// hooks/useOfflineSync.ts — flush queue when back online
export function useOfflineSync() {
  const { queue, removeFromQueue, incrementRetry } = useOfflineQueue();
  const isOnline = useNetworkStatus();

  useEffect(() => {
    if (!isOnline || queue.length === 0) return;

    async function flushQueue() {
      for (const mutation of queue) {
        try {
          await processMutation(mutation);
          removeFromQueue(mutation.id);
        } catch (error) {
          if (mutation.retryCount >= 3) {
            // Give up after 3 retries — remove and notify user
            removeFromQueue(mutation.id);
            showToast({ type: 'error', message: `Failed to sync: ${mutation.type}` });
          } else {
            incrementRetry(mutation.id);
          }
        }
      }
    }

    flushQueue();
  }, [isOnline]);
}

async function processMutation(mutation: PendingMutation) {
  switch (mutation.type) {
    case 'SEND_MESSAGE':
      return api.sendMessage(mutation.payload);
    case 'SAVE_DRAFT':
      return api.saveDraft(mutation.payload);
    case 'LIKE_POST':
      return api.likePost(mutation.payload);
    default:
      throw new Error(`Unknown mutation type: ${mutation.type}`);
  }
}
```

### Using the offline queue

```tsx
function SendMessageButton({ content }: { content: string }) {
  const { addToQueue } = useOfflineQueue();
  const isOnline = useNetworkStatus();
  const mutation = useMutation({ mutationFn: api.sendMessage });

  async function handleSend() {
    if (isOnline) {
      // Online: send immediately
      mutation.mutate({ content });
    } else {
      // Offline: queue for later
      addToQueue({ type: 'SEND_MESSAGE', payload: { content } });
      // Show optimistic UI — message appears as "sending"
      addOptimisticMessage({ content, status: 'pending' });
      showToast({ message: "Message queued — will send when back online" });
    }
  }
}
```

---

## OFFLINE-FIRST DATA PATTERNS

### Local-first with Expo SQLite

For apps where offline is the primary use case (POS system, field work, notes):

```tsx
// Always write to SQLite first, sync to server in background
async function saveOrder(order: Order) {
  // 1. Write locally — instant, always works
  await db.runAsync(
    'INSERT INTO orders (id, data, synced) VALUES (?, ?, 0)',
    [order.id, JSON.stringify(order)]
  );

  // 2. Show optimistic UI immediately
  setOrders(prev => [...prev, order]);

  // 3. Sync to server in background (don't await)
  syncOrderToServer(order).catch(() => {
    // Failed — will retry when online
    markForSync(order.id);
  });
}

// Sync loop — runs whenever app comes online
async function syncPendingOrders() {
  const unsynced = await db.getAllAsync(
    'SELECT * FROM orders WHERE synced = 0'
  );

  for (const order of unsynced) {
    try {
      await api.createOrder(JSON.parse(order.data));
      await db.runAsync('UPDATE orders SET synced = 1 WHERE id = ?', [order.id]);
    } catch (error) {
      // Leave synced = 0, will try again next time
    }
  }
}
```

---

## NETWORK ERROR HANDLING

### Classify errors before showing them to users

```tsx
// lib/errors.ts
export function classifyNetworkError(error: unknown): {
  type: 'offline' | 'timeout' | 'server' | 'auth' | 'client';
  message: string;
  retryable: boolean;
} {
  if (!navigator.onLine) {
    return { type: 'offline', message: 'No internet connection', retryable: true };
  }

  if (axios.isAxiosError(error)) {
    const status = error.response?.status;

    if (!error.response) {
      return { type: 'timeout', message: 'Request timed out', retryable: true };
    }
    if (status === 401) {
      return { type: 'auth', message: 'Please sign in again', retryable: false };
    }
    if (status === 403) {
      return { type: 'client', message: "You don't have permission", retryable: false };
    }
    if (status === 404) {
      return { type: 'client', message: 'Not found', retryable: false };
    }
    if (status && status >= 500) {
      return { type: 'server', message: "Server error — we're working on it", retryable: true };
    }
  }

  return { type: 'server', message: 'Something went wrong', retryable: true };
}
```

### Exponential backoff for retries

```tsx
// Retry with exponential backoff: 1s → 2s → 4s → 8s
async function retryWithBackoff<T>(
  fn: () => Promise<T>,
  maxRetries = 4
): Promise<T> {
  let lastError: unknown;

  for (let attempt = 0; attempt < maxRetries; attempt++) {
    try {
      return await fn();
    } catch (error) {
      lastError = error;
      const delay = Math.min(1000 * Math.pow(2, attempt), 30000);
      await new Promise(resolve => setTimeout(resolve, delay));
    }
  }

  throw lastError;
}
```

---

## OFFLINE CHECKLIST

```
Detection:
- [ ] NetInfo installed and configured
- [ ] useNetworkStatus hook created
- [ ] OfflineBanner component wired to root layout
- [ ] isInternetReachable checked (not just isConnected)

Data:
- [ ] TanStack Query networkMode: 'offlineFirst'
- [ ] staleTime + gcTime set (data survives app restart with persister)
- [ ] Query cache persisted to MMKV (survives app restarts)
- [ ] Critical data stored in Expo SQLite for offline access

Mutations:
- [ ] Mutations that matter have offline queue fallback
- [ ] Offline queue persisted to MMKV
- [ ] Queue flushes automatically when back online
- [ ] User notified when action is queued vs completed

Errors:
- [ ] Network errors classified (offline / timeout / server / auth / client)
- [ ] Error messages are actionable (retry button on retryable errors)
- [ ] Retry uses exponential backoff
- [ ] Sentry logs all non-retryable errors
```
