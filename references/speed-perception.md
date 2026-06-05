# Speed Perception — Making Your App Feel Fast (June 2026)

**Also use:** TanStack Query `placeholderData`, expo-image blurhash, React Compiler, FlashList v2 / Legend List v3.

## THE FUNDAMENTAL TRUTH

A 500ms action that feels instant beats a 100ms action that feels slow.
Perceived speed ≠ actual speed.
Engineer both.

```
Actual speed      → covered in performance.md (FPS, cold start, bundle size)
Perceived speed   → covered HERE (what users feel, not what Xcode measures)
```

---

## THE 3 PERCEPTION THRESHOLDS

```
< 100ms   → feels instant (no feedback needed)
100–300ms → feels immediate (subtle animation acceptable)
300–1000ms → feels slow (show activity indicator or optimistic UI)
> 1000ms  → feels broken (progress bar, skeleton, or cancel option required)
```

Design every interaction to feel like it's in the first two buckets,
even when the actual server round-trip is in the third or fourth.

---

## COLD START PERCEPTION

Cold start = time from tap to first meaningful content.
Users form their first impression here. You have ~2 seconds before they assume the app is broken.

### The perception sequence
```
0ms     → App icon pressed (haptic if supported)
0-200ms → Splash screen (instant — hardcoded, zero network)
200ms   → Shell renders (nav bar, tab bar, placeholder layout)
400ms   → Stale/cached content appears (from MMKV or TanStack Query cache)
800ms   → Fresh data replaces stale (background refetch completes)
```

The key insight: **users see content at 400ms**, not when fresh data arrives at 800ms.
The 400ms → 800ms window is silent background refresh — users don't know it's happening.

### Implementation
```tsx
// TanStack Query: show stale data immediately
const { data } = useQuery({
  queryKey: ['home-feed'],
  queryFn: fetchFeed,
  staleTime: 60_000,        // data is "fresh" for 60s — no refetch in this window
  gcTime: 5 * 60_000,       // keep in cache for 5 minutes after unmount
  placeholderData: keepPreviousData,  // show previous data while new data loads
});

// MMKV: persist critical data between sessions
// On app open: data is available before any network call
const cachedUser = storage.getString('current_user');
if (cachedUser) {
  setUser(JSON.parse(cachedUser));  // instant — no network
}
```

---

## NAVIGATION SPEED PERCEPTION

### Pre-fetch on intent (not on tap)
Users press down before they press up. You have 80-120ms head start.

```tsx
<Pressable
  // User touches the button — start fetching immediately
  onPressIn={() => {
    queryClient.prefetchQuery({
      queryKey: ['post', item.id],
      queryFn: () => api.getPost(item.id),
      staleTime: 10_000,
    });
  }}
  // User releases — data may already be cached
  onPress={() => router.push(`/post/${item.id}`)}
>
  <PostCard item={item} />
</Pressable>
```

### Pre-fetch next page before user reaches it
```tsx
<FlashList
  data={items}
  onEndReached={() => {
    if (hasNextPage) fetchNextPage();
  }}
  onEndReachedThreshold={0.5}  // start loading when 50% from bottom
  // Pre-render items beyond viewport
  drawDistance={500}           // render 500px below visible area
/>
```

### Instant tab switching with pre-rendered screens
```tsx
// Keep screens mounted when switching tabs (don't unmount/remount)
// expo-router/unstable-native-tabs does this automatically
// For standard Tabs, use detachInactiveScreens={false}:
<Tabs screenOptions={{ detachInactiveScreens: false }}>
```

---

## LOADING STATE PSYCHOLOGY

### The loading state hierarchy (from best to worst perceived speed)

```
1. No loading state (optimistic UI)          → feels instant
2. Instant stale content + silent refresh    → feels instant
3. Skeleton with correct layout              → feels fast
4. Skeleton with wrong layout                → feels slow (layout shift)
5. Spinner                                   → feels slow
6. Blank screen                              → feels broken
```

### Progressive content loading — above-fold first
```tsx
function ArticleScreen({ id }: { id: string }) {
  // Load header data first (small payload, fast)
  const { data: header } = useQuery({
    queryKey: ['article-header', id],
    queryFn: () => api.getArticleHeader(id),
  });

  // Load body content separately (large payload, slower)
  const { data: body } = useQuery({
    queryKey: ['article-body', id],
    queryFn: () => api.getArticleBody(id),
    enabled: !!header,  // only start after header loads
  });

  return (
    <ScrollView>
      {/* Header renders immediately */}
      {header
        ? <ArticleHeader data={header} />
        : <ArticleHeaderSkeleton />
      }

      {/* Body follows */}
      {body
        ? <ArticleBody data={body} />
        : <ArticleBodySkeleton />
      }
    </ScrollView>
  );
}
```

### The 200ms rule for loading indicators
Don't show a spinner for actions that usually complete in < 200ms.
Showing a spinner for a fast action makes it feel slower.

```tsx
function SmartLoadingButton({ onPress }: { onPress: () => Promise<void> }) {
  const [isLoading, setIsLoading] = useState(false);
  const [showSpinner, setShowSpinner] = useState(false);

  async function handlePress() {
    setIsLoading(true);

    // Only show spinner if still loading after 200ms
    const spinnerTimer = setTimeout(() => setShowSpinner(true), 200);

    try {
      await onPress();
    } finally {
      clearTimeout(spinnerTimer);
      setIsLoading(false);
      setShowSpinner(false);
    }
  }

  return (
    <Pressable onPress={handlePress} disabled={isLoading}>
      {showSpinner ? <ActivityIndicator /> : <Text>Submit</Text>}
    </Pressable>
  );
}
```

---

## ANIMATION SPEED PSYCHOLOGY

### Duration guidelines

| Interaction | Duration | Feel |
|---|---|---|
| Button press scale | 80–100ms | Instant response |
| Tab switch | 250–300ms | Smooth but quick |
| Modal appear | 300–350ms | Natural |
| Modal dismiss | 200–250ms | Slightly faster than appear |
| Page transition | 350–400ms | Navigation feels intentional |
| Success celebration | 500–600ms | Memorable but not annoying |
| Onboarding screens | 400–500ms | Deliberate, premium |

**Rule:** Entrance animations are slower than exit animations.
Things entering your world take time. Things leaving get out of the way quickly.

### The Anticipation Principle
Users feel faster response when the UI reacts before the action completes.
```tsx
// ❌ No anticipation — button does nothing until tap completes
<Pressable onPress={handleSubmit}>
  <Text>Submit</Text>
</Pressable>

// ✅ Immediate visual response on press start
const scale = useSharedValue(1);
<GestureDetector gesture={
  Gesture.Tap()
    .onBegin(() => { scale.value = withSpring(0.96) })  // immediate
    .onFinalize(() => { scale.value = withSpring(1) })
}>
```

### Easing curves and their psychological effects

```tsx
import { Easing } from 'react-native-reanimated';

// Ease out — starts fast, ends slow — feels natural for things entering
withTiming(value, { easing: Easing.out(Easing.cubic) })

// Ease in — starts slow, ends fast — feels natural for things exiting
withTiming(value, { easing: Easing.in(Easing.cubic) })

// Ease in-out — starts slow, fast middle, ends slow — navigation transitions
withTiming(value, { easing: Easing.inOut(Easing.cubic) })

// Spring — physical, bouncy — interactions, feedback
withSpring(value, { damping: 15, stiffness: 150 })

// Linear — mechanical, robotic — avoid for UI (use for progress bars only)
withTiming(value, { easing: Easing.linear })
```

---

## SCROLL PERFORMANCE PERCEPTION

### The blank flash problem
A flash of white/blank content during fast scroll = app feels slow even if FPS is perfect.
Cause: JS thread can't render new cells fast enough.
Fix: LegendList (JSI-based, renders ahead of scroll natively).

### Pull-to-refresh psychology
```tsx
// Haptic on threshold — confirms to user they've pulled enough
<ScrollView
  refreshControl={
    <RefreshControl
      refreshing={isRefreshing}
      onRefresh={async () => {
        ReactNativeHaptics.impact('medium');  // haptic when refresh triggers
        await refetch();
      }}
      tintColor="#0ea5e9"  // match brand color
    />
  }
/>
```

### Infinite scroll — start loading early
```tsx
// Start loading next page when user is 50% from bottom, not at bottom
// By the time user reaches bottom, data is already loaded
onEndReachedThreshold={0.5}

// Pair with FlashList drawDistance to pre-render cells
drawDistance={600}  // render 600px below visible area
```

---

## NETWORK SPEED PERCEPTION

### Request deduplication (TanStack Query automatic)
Multiple components requesting the same data = one network call.
TanStack Query deduplicates automatically within the same query key.

### Request racing — cancel stale requests
```tsx
const { data } = useQuery({
  queryKey: ['search', query],
  queryFn: ({ signal }) => api.search(query, { signal }),  // pass AbortSignal
  enabled: query.length > 2,  // don't fetch for tiny queries
});
// If query changes before response: previous request is automatically cancelled
```

### Background sync — update without user knowing
```tsx
// Refetch in background when app comes to foreground
const { data } = useQuery({
  queryKey: ['notifications'],
  queryFn: fetchNotifications,
  refetchOnWindowFocus: true,     // refetch when app comes to foreground
  refetchInterval: 60_000,        // also poll every 60s
  refetchIntervalInBackground: false,  // don't poll when app is backgrounded
});
```

---

## IMAGE LOADING PERCEPTION

### Blurhash — instant placeholder, no layout shift
```tsx
import { Image } from 'expo-image';

<Image
  source={{ uri: photo.url }}
  placeholder={{ blurhash: photo.blurhash }}  // generated server-side, stored in DB
  transition={150}  // fade in over 150ms — fast enough to feel instant
  contentFit="cover"
/>
```

Generate blurhash server-side when images are uploaded:
```bash
npm install blurhash  # server-side
```

```ts
// server — when user uploads image
import { encode } from 'blurhash';
const blurhash = encode(pixels, width, height, 4, 3);
// Store blurhash alongside image URL in database
```

### Progressive image loading
```tsx
// Show low-quality thumbnail first, then full quality
<Image
  source={{ uri: photo.fullUrl }}
  placeholder={{ uri: photo.thumbnailUrl }}  // tiny image (< 1KB) loads instantly
  transition={200}
  contentFit="cover"
/>
```

---

## PERCEIVED SPEED CHECKLIST

```
Cold start:
- [ ] Splash screen is instant (no network calls before it shows)
- [ ] Shell/layout renders before data loads
- [ ] Cached data visible within 400ms of app open
- [ ] Fresh data replaces stale silently (no flash)

Navigation:
- [ ] onPressIn pre-fetches target screen data
- [ ] Tab screens stay mounted (detachInactiveScreens: false)
- [ ] Next page pre-fetched before user reaches bottom

Loading states:
- [ ] Skeletons match actual content layout (no layout shift)
- [ ] No spinners for actions < 200ms
- [ ] Above-fold content loads before below-fold

Animations:
- [ ] Button press response < 100ms
- [ ] All animations use correct easing (ease-out for enter, ease-in for exit)
- [ ] Haptic fires on press, not on action completion

Images:
- [ ] Blurhash placeholders for all remote images
- [ ] expo-image used for all remote images (not built-in Image)
- [ ] Images cached (cachePolicy="memory-disk")

Network:
- [ ] TanStack Query deduplicates parallel requests
- [ ] Search queries cancelled on keystroke
- [ ] App refetches on foreground (refetchOnWindowFocus: true)
```
