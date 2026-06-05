# Data & State — TanStack Query, Zustand, Forms, SQLite (June 2026)

## THE STATE SPLIT

```
Server data (from API)       → TanStack Query
UI / app state               → Zustand
Persistent non-secret state  → Zustand + MMKV v4 persist
Auth tokens / secrets        → expo-secure-store only
Form state                   → React Hook Form + Zod
Local relational data        → expo-sqlite (WAL)
HTTP transport               → fetch (expo/fetch default SDK 56) or Axios
```

Never use one solution for everything. This split eliminates 90% of state management bugs.

---

## HTTP — `expo/fetch` (SDK 56)

`expo/fetch` is **`globalThis.fetch` by default** — WinterTC-compliant, compression on Android.

```ts
// Works without manual import in SDK 56+
const res = await fetch(`${process.env.EXPO_PUBLIC_API_URL}/users`);
```

Opt out for legacy tooling: `EXPO_PUBLIC_USE_RN_FETCH=1` in `.env`

**TanStack Query** works with global fetch — no Axios required.

**Sentry (pick one strategy — not both blindly):**

| Fetch mode | Sentry |
|---|---|
| Default `expo/fetch` | `traceFetch: true` if HTTP spans missing |
| `EXPO_PUBLIC_USE_RN_FETCH=1` | **`traceFetch: false`** — avoid duplicate spans with RN fetch |

See `references/production-edge-cases.md` for conditional `Sentry.init`.

Axios remains valid for interceptors/interop — pair with TanStack Query either way.

---

## TANSTACK QUERY — SERVER STATE

```bash
npm install @tanstack/react-query axios
```

### Setup
```tsx
// app/_layout.tsx
import { QueryClient, QueryClientProvider } from '@tanstack/react-query';

const queryClient = new QueryClient({
  defaultOptions: {
    queries: {
      retry: 2,
      staleTime: 1000 * 60 * 5,  // 5 minutes
      gcTime: 1000 * 60 * 10,    // 10 minutes
    },
  },
});

export default function RootLayout() {
  return (
    <QueryClientProvider client={queryClient}>
      <GestureHandlerRootView style={{ flex: 1 }}>
        <Stack />
      </GestureHandlerRootView>
    </QueryClientProvider>
  );
}
```

### Basic query
```tsx
import { useQuery } from '@tanstack/react-query';
import axios from 'axios';

function UserProfile({ userId }: { userId: string }) {
  const { data, isLoading, isError, error, refetch } = useQuery({
    queryKey: ['user', userId],
    queryFn: async () => {
      const { data } = await axios.get(`/api/users/${userId}`);
      return data;
    },
  });

  if (isLoading) return <UserSkeleton />;
  if (isError) return <ErrorView error={error} onRetry={refetch} />;

  return <UserCard user={data} />;
}
```

### Infinite scroll (feeds, lists)
```tsx
import { useInfiniteQuery } from '@tanstack/react-query';

function PostsFeed() {
  const {
    data,
    fetchNextPage,
    hasNextPage,
    isFetchingNextPage,
  } = useInfiniteQuery({
    queryKey: ['posts'],
    queryFn: async ({ pageParam = 1 }) => {
      const { data } = await axios.get(`/api/posts?page=${pageParam}`);
      return data;
    },
    getNextPageParam: (lastPage) => lastPage.nextPage ?? undefined,
    initialPageParam: 1,
  });

  const allPosts = data?.pages.flatMap((page) => page.items) ?? [];

  return (
    <FlashList
      data={allPosts}
      renderItem={({ item }) => <PostCard post={item} />}
      onEndReached={() => hasNextPage && fetchNextPage()}
      onEndReachedThreshold={0.5}
      ListFooterComponent={isFetchingNextPage ? <LoadingSpinner /> : null}
    />
  );
  // FlashList v2: no estimatedItemSize — New Arch only
}
```

### Mutations (create, update, delete)
```tsx
import { useMutation, useQueryClient } from '@tanstack/react-query';

function CreatePost() {
  const queryClient = useQueryClient();

  const mutation = useMutation({
    mutationFn: (newPost: { title: string; body: string }) =>
      axios.post('/api/posts', newPost),
    onSuccess: () => {
      // Invalidate and refetch posts list
      queryClient.invalidateQueries({ queryKey: ['posts'] });
    },
    onError: (error) => {
      // Show error toast
    },
  });

  return (
    <Button
      onPress={() => mutation.mutate({ title: 'Hello', body: 'World' })}
      loading={mutation.isPending}
    />
  );
}
```

### Optimistic updates (instant UI feedback)
```tsx
const mutation = useMutation({
  mutationFn: (postId: string) => axios.post(`/api/posts/${postId}/like`),
  onMutate: async (postId) => {
    // Cancel in-flight refetches
    await queryClient.cancelQueries({ queryKey: ['posts'] });
    // Snapshot current state
    const previousPosts = queryClient.getQueryData(['posts']);
    // Optimistically update
    queryClient.setQueryData(['posts'], (old: Post[]) =>
      old.map(p => p.id === postId ? { ...p, liked: true, likeCount: p.likeCount + 1 } : p)
    );
    return { previousPosts };
  },
  onError: (err, postId, context) => {
    // Rollback on error
    queryClient.setQueryData(['posts'], context?.previousPosts);
  },
});
```

---

## AXIOS SETUP — HTTP LAYER

```tsx
// services/api.ts
import axios from 'axios';
import { storage } from '../lib/storage';  // MMKV v4 — never store auth tokens here

export const api = axios.create({
  baseURL: process.env.EXPO_PUBLIC_API_URL,
  timeout: 10000,
  headers: {
    'Content-Type': 'application/json',
  },
});

// Auth token interceptor
api.interceptors.request.use((config) => {
  const token = storage.getString('auth.token');
  if (token) {
    config.headers.Authorization = `Bearer ${token}`;
  }
  return config;
});

// Global error handler
api.interceptors.response.use(
  (response) => response,
  (error) => {
    if (error.response?.status === 401) {
      // Token expired — clear auth and redirect to login
      storage.delete('auth.token');
      // router.replace('/login');  // navigate if needed
    }
    return Promise.reject(error);
  }
);
```

---

## ZUSTAND — CLIENT STATE

```bash
npm install zustand
```

### Basic store
```tsx
// stores/ui.store.ts
import { create } from 'zustand';

interface UIStore {
  isBottomSheetOpen: boolean;
  selectedTab: string;
  openBottomSheet: () => void;
  closeBottomSheet: () => void;
  setSelectedTab: (tab: string) => void;
}

export const useUIStore = create<UIStore>((set) => ({
  isBottomSheetOpen: false,
  selectedTab: 'home',
  openBottomSheet: () => set({ isBottomSheetOpen: true }),
  closeBottomSheet: () => set({ isBottomSheetOpen: false }),
  setSelectedTab: (tab) => set({ selectedTab: tab }),
}));
```

### Usage
```tsx
// Only re-renders when isBottomSheetOpen changes (selector pattern)
const isOpen = useUIStore((state) => state.isBottomSheetOpen);
const openSheet = useUIStore((state) => state.openBottomSheet);
```

### Persistent store (auth, settings, user preferences)
```tsx
// stores/auth.store.ts
import { create } from 'zustand';
import { persist, createJSONStorage } from 'zustand/middleware';
import { storage } from '../lib/storage';

const mmkvStorage = {
  getItem: (key: string) => storage.getString(key) ?? null,
  setItem: (key: string, value: string) => storage.set(key, value),
  removeItem: (key: string) => storage.delete(key),
};

interface AuthStore {
  token: string | null;
  user: { id: string; name: string; email: string } | null;
  isAuthenticated: boolean;
  login: (token: string, user: AuthStore['user']) => void;
  logout: () => void;
}

export const useAuthStore = create<AuthStore>()(
  persist(
    (set) => ({
      token: null,
      user: null,
      isAuthenticated: false,
      login: (token, user) => set({ token, user, isAuthenticated: true }),
      logout: () => set({ token: null, user: null, isAuthenticated: false }),
    }),
    {
      name: 'auth',
      storage: createJSONStorage(() => mmkvStorage),
    }
  )
);
```

---

## REACT HOOK FORM + ZOD — FORMS

```bash
npm install react-hook-form zod @hookform/resolvers
```

```tsx
import { useForm, Controller } from 'react-hook-form';
import { z } from 'zod';
import { zodResolver } from '@hookform/resolvers/zod';

// Define schema with Zod
const loginSchema = z.object({
  email: z.string().email('Invalid email'),
  password: z.string().min(8, 'Password must be at least 8 characters'),
});

type LoginForm = z.infer<typeof loginSchema>;

function LoginScreen() {
  const {
    control,
    handleSubmit,
    formState: { errors, isSubmitting },
  } = useForm<LoginForm>({
    resolver: zodResolver(loginSchema),
    defaultValues: { email: '', password: '' },
  });

  const onSubmit = async (data: LoginForm) => {
    // data is fully typed and validated
    await login(data.email, data.password);
  };

  return (
    <View>
      <Controller
        control={control}
        name="email"
        render={({ field: { onChange, value } }) => (
          <TextInput
            value={value}
            onChangeText={onChange}
            placeholder="Email"
            keyboardType="email-address"
            autoCapitalize="none"
            className={cn(
              "border rounded-xl px-4 py-3",
              errors.email ? "border-red-500" : "border-gray-300"
            )}
          />
        )}
      />
      {errors.email && (
        <Text className="text-red-500 text-sm mt-1">{errors.email.message}</Text>
      )}

      <Controller
        control={control}
        name="password"
        render={({ field: { onChange, value } }) => (
          <TextInput
            value={value}
            onChangeText={onChange}
            placeholder="Password"
            secureTextEntry
            className="border border-gray-300 rounded-xl px-4 py-3"
          />
        )}
      />

      <Button
        onPress={handleSubmit(onSubmit)}
        loading={isSubmitting}
        title="Log In"
      />
    </View>
  );
}
```

**Why React Hook Form:** zero re-renders on keystroke (watches field state internally,
not React state). For a form with 10 fields, this means 10x fewer renders vs useState.

---

## ENVIRONMENT VARIABLES

```bash
# .env.local
EXPO_PUBLIC_API_URL=https://api.myapp.com
EXPO_PUBLIC_SUPABASE_URL=https://xxx.supabase.co
EXPO_PUBLIC_SUPABASE_ANON_KEY=your-anon-key
EXPO_PUBLIC_POSTHOG_KEY=your-key
```

Access in code:
```tsx
const apiUrl = process.env.EXPO_PUBLIC_API_URL;
```

**EXPO_PUBLIC_ prefix = exposed to JS bundle (client-side safe).**
Never put secret keys with EXPO_PUBLIC_ — they're in the app bundle.
Secret keys go in EAS Secrets (server/build-time only).

---

## SUPABASE INTEGRATION

```bash
npm install @supabase/supabase-js
npx expo install expo-secure-store
```

```tsx
// lib/supabase.ts
import { createClient } from '@supabase/supabase-js';
import * as SecureStore from 'expo-secure-store';

const ExpoSecureStoreAdapter = {
  getItem: (key: string) => SecureStore.getItemAsync(key),
  setItem: (key: string, value: string) => SecureStore.setItemAsync(key, value),
  removeItem: (key: string) => SecureStore.deleteItemAsync(key),
};

export const supabase = createClient(
  process.env.EXPO_PUBLIC_SUPABASE_URL!,
  process.env.EXPO_PUBLIC_SUPABASE_ANON_KEY!,
  {
    auth: {
      storage: ExpoSecureStoreAdapter,
      autoRefreshToken: true,
      persistSession: true,
      detectSessionInUrl: false,
    },
  }
);
```

Use `expo-secure-store` for Supabase auth tokens — encrypted, keychain-backed.
Don't use MMKV for auth tokens — not encrypted by default.
