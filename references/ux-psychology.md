# UX Psychology — How Users Feel (June 2026)

Visual clutter and generic “AI app” layouts: see `anti-ai-slop.md` (card nesting, color traps, copy).

Use React 19 `useOptimistic()` with TanStack Query `onMutate` rollback for server-backed actions.

## THE CORE TRUTH

Users don't experience your app's actual performance.
They experience their *perception* of your app's performance.
A 200ms response that feels instant beats a 100ms response that feels slow.
Engineering perception is as important as engineering speed.

---

## THE 4 PSYCHOLOGICAL LAWS THAT GOVERN MOBILE UX

### 1. Fitts's Law — Touch Targets
The time to hit a target = distance to target / size of target.

**Rules:**
- Minimum touch target: **44×44pt** (Apple HIG) / **48×48dp** (Material)
- Primary actions (CTA, submit): 56pt+ height
- Destructive actions (delete): deliberately harder to reach — bottom of screen, requires confirmation
- Navigation items: full tab bar height, not just the icon

```tsx
// ❌ Too small
<Pressable style={{ width: 20, height: 20 }}>
  <Icon size={16} />
</Pressable>

// ✅ Correct — visual size 20pt, hit area 44pt
<Pressable
  style={{ padding: 12 }}  // 12+20+12 = 44pt total
  hitSlop={{ top: 8, bottom: 8, left: 8, right: 8 }}  // extra insurance
>
  <Icon size={20} />
</Pressable>
```

### 2. Miller's Law — Cognitive Load
Users can hold 7±2 items in working memory at once.

**Rules:**
- Max 5 tab bar items (ideally 3-4)
- Max 7 items in a menu or picker
- Group related actions — never show 12 options at once
- Progressive disclosure: show details only when needed

```tsx
// ❌ Cognitive overload
<ActionSheet>
  {twelveOptions.map(option => <Option key={option.id} />)}
</ActionSheet>

// ✅ Grouped, progressive
<ActionSheet>
  <PrimaryActions />  {/* 3 most common */}
  <Pressable onPress={showMore}>More options...</Pressable>
</ActionSheet>
```

### 3. Jakob's Law — Familiarity
Users spend most of their time on other apps.
They expect your app to work like apps they already know.

**Rules:**
- iOS: follow Apple HIG conventions (back swipe, tab bar position, share sheet)
- Android: follow Material 3 (FAB position, bottom nav, predictive back)
- Don't invent navigation patterns — users pay a learning tax on every novel pattern
- Pull-to-refresh, swipe-to-delete, long-press for context menu — these are expected

### 4. Peak-End Rule — Memory
Users remember an experience by its peak moment and its end.
The middle doesn't matter as much as you think.

**Rules:**
- Make the first successful action memorable (celebrate it with haptics + animation)
- Make the last action before exit feel complete and resolved
- Fix the worst moment in your app (longest load, biggest error) — that's what users remember
- Empty states and error states ARE your peak-end moments for new users

---

## PERCEIVED PERFORMANCE PATTERNS

### Optimistic UI — the most impactful pattern
Show the result immediately. Confirm with server in background. Rollback on error.

```tsx
// ❌ Wait for server — feels slow
async function handleLike() {
  setLoading(true);
  await api.likePost(postId);  // 300ms wait
  setIsLiked(true);
  setLoading(false);
}

// ✅ Optimistic — feels instant
function handleLike() {
  // Show result immediately
  setIsLiked(true);
  setLikeCount(c => c + 1);
  triggerHaptic('light');

  // Confirm in background
  api.likePost(postId).catch(() => {
    // Silent rollback
    setIsLiked(false);
    setLikeCount(c => c - 1);
  });
}
```

Use `useOptimistic()` from React 19 for cleaner pattern:
```tsx
const [optimisticLiked, setOptimisticLiked] = useOptimistic(isLiked);

function handleLike() {
  setOptimisticLiked(!isLiked);  // instant UI update
  startTransition(() => {
    likePost(postId);  // async confirm
  });
}
```

### Instant Navigation — pre-fetch on hover/focus
```tsx
// Pre-fetch data when user starts navigating toward a screen
// (touch down, not touch up — 80ms head start)
<Pressable
  onPressIn={() => queryClient.prefetchQuery({
    queryKey: ['post', item.id],
    queryFn: () => api.getPost(item.id),
  })}
  onPress={() => router.push(`/post/${item.id}`)}
>
```

### Loading Order Psychology
Users perceive apps as faster when content loads in the right order:

```
1. Layout/shell (instant — hardcoded, no data)
2. Cached/stale data (instant — from MMKV or TanStack Query cache)
3. Critical above-fold content (first)
4. Below-fold content (lazy)
5. Non-essential metadata (last)
```

```tsx
// Show stale data immediately, update silently
const { data } = useQuery({
  queryKey: ['feed'],
  queryFn: fetchFeed,
  staleTime: 30_000,        // Show cached version for 30s without refetch indicator
  placeholderData: keepPreviousData,  // Keep old data visible during refresh
});
```

### Skeleton vs Spinner Psychology
- **Skeleton:** communicates structure, feels faster, reduces layout shift
- **Spinner:** communicates "I don't know how long this will take"
- Rule: always skeleton for content, spinner only for actions (button loading state)

```tsx
// Loading state hierarchy
if (isInitialLoading) return <ScreenSkeleton />;  // first load
if (isError) return <ErrorView onRetry={refetch} />;
if (!data) return <EmptyState />;

// Data is ready — render with possible background refresh indicator
return (
  <>
    {isFetching && !isInitialLoading && <RefreshIndicator />}
    <ContentList data={data} />
  </>
);
```

---

## EMPTY STATES — YOUR CONVERSION MOMENT

Empty states are not edge cases. They are the first thing new users see.
A bad empty state = churn. A good empty state = activation.

```tsx
// ❌ Lazy empty state
if (!items.length) return <Text>No items found</Text>;

// ✅ Conversion-focused empty state
function EmptyFeed() {
  return (
    <View className="flex-1 items-center justify-center p-8">
      {/* Illustration — Lottie or static SVG */}
      <LottieView source={require('./empty-feed.json')} autoPlay loop={false} />

      {/* Clear value proposition */}
      <Text className="text-2xl font-bold text-center mt-6">
        Your feed is empty
      </Text>
      <Text className="text-gray-500 text-center mt-2">
        Follow people or topics to see posts here
      </Text>

      {/* Single clear CTA */}
      <Pressable
        className="bg-blue-500 rounded-full px-8 py-4 mt-8"
        onPress={() => router.push('/discover')}
      >
        <Text className="text-white font-semibold">Discover people</Text>
      </Pressable>
    </View>
  );
}
```

**Empty state checklist:**
- [ ] Illustration (not a generic icon)
- [ ] Clear explanation of WHY it's empty
- [ ] Single action to fix it (not three options)
- [ ] Friendly, not apologetic tone

---

## ERROR STATES — RECOVERY, NOT BLAME

```tsx
// ❌ Generic error
<Text>Something went wrong</Text>

// ✅ Actionable error
function ErrorView({ error, onRetry }: { error: Error; onRetry: () => void }) {
  const isNetworkError = !navigator.onLine;

  return (
    <View className="flex-1 items-center justify-center p-8">
      <SymbolView name={isNetworkError ? "wifi.slash" : "exclamationmark.circle"} size={48} />

      <Text className="text-xl font-bold mt-4">
        {isNetworkError ? "No internet connection" : "Couldn't load content"}
      </Text>
      <Text className="text-gray-500 text-center mt-2">
        {isNetworkError
          ? "Check your connection and try again"
          : "This is on our end. We're looking into it."}
      </Text>

      <Pressable
        className="bg-gray-900 rounded-full px-8 py-4 mt-6"
        onPress={onRetry}
      >
        <Text className="text-white font-semibold">Try again</Text>
      </Pressable>
    </View>
  );
}
```

**Error state rules:**
- Never say "Something went wrong" — say what went wrong
- Never blame the user — say what YOU will do
- Always provide one clear recovery action
- Network errors and server errors need different messages
- Log every error to Sentry silently

---

## FORM UX PSYCHOLOGY

### Reduce friction at every step
```tsx
// Keyboard types reduce typing effort
<TextInput keyboardType="email-address" autoCapitalize="none" autoCorrect={false} />
<TextInput keyboardType="phone-pad" />
<TextInput keyboardType="decimal-pad" />
<TextInput secureTextEntry textContentType="password" />  // enables password autofill

// textContentType enables iOS AutoFill
<TextInput textContentType="emailAddress" />
<TextInput textContentType="newPassword" />  // triggers strong password suggestion
<TextInput textContentType="oneTimeCode" />  // auto-reads SMS verification codes
```

### Inline validation — don't wait for submit
```tsx
// Validate on blur, not on change (validating while typing = frustrating)
<Controller
  name="email"
  control={control}
  render={({ field }) => (
    <TextInput
      {...field}
      onBlur={() => {
        field.onBlur();
        trigger('email');  // validate only after user leaves the field
      }}
    />
  )}
/>
```

### Submit button state communicates progress
```tsx
function SubmitButton({ isLoading, isSuccess }: { isLoading: boolean; isSuccess: boolean }) {
  return (
    <MotiView animate={{ scale: isSuccess ? [1, 1.05, 1] : 1 }}>
      <Pressable
        disabled={isLoading || isSuccess}
        className={cn(
          "rounded-full py-4 items-center",
          isLoading && "bg-gray-400",
          isSuccess && "bg-green-500",
          !isLoading && !isSuccess && "bg-blue-500"
        )}
      >
        {isLoading && <ActivityIndicator color="white" />}
        {isSuccess && <SymbolView name="checkmark" tintColor="white" />}
        {!isLoading && !isSuccess && <Text className="text-white font-bold">Submit</Text>}
      </Pressable>
    </MotiView>
  );
}
```

---

## ONBOARDING PSYCHOLOGY

### Progressive permission requests
Never ask for all permissions on first launch. Ask in context, when the user understands the value.

```
❌ App opens → "Allow notifications?" → user taps No → never gets notifications

✅ User completes first action → "Turn on notifications to know when X happens?" →
   user says Yes because they understand the value now
```

```tsx
// Ask for permissions at the moment of value, not at launch
async function handleFollowUser(userId: string) {
  await api.follow(userId);

  // NOW ask for notifications — user just demonstrated they care about this person
  const { status } = await Notifications.getPermissionsAsync();
  if (status === 'undetermined') {
    showNotificationPrompt();  // your custom explanation screen
  }
}
```

### The 3-screen onboarding rule
Users skip onboarding. If you must have it:
- Max 3 screens
- Show value, not features ("See what your friends are up to" not "Our feed algorithm")
- Skip button always visible
- Last screen = immediate activation (not "you're all set, go explore")

---

## MICRO-INTERACTION PATTERNS

These are the details that make an app feel alive:

### Button press feedback
```tsx
function PressableButton({ onPress, children }: ButtonProps) {
  const scale = useSharedValue(1);

  const animatedStyle = useAnimatedStyle(() => ({
    transform: [{ scale: scale.value }],
  }));

  return (
    <GestureDetector gesture={
      Gesture.Tap()
        .onBegin(() => {
          scale.value = withSpring(0.96, { damping: 15 });
          runOnJS(ReactNativeHaptics.impact)('light');
        })
        .onFinalize(() => {
          scale.value = withSpring(1, { damping: 15 });
          runOnJS(onPress)();
        })
    }>
      <Animated.View style={animatedStyle}>
        {children}
      </Animated.View>
    </GestureDetector>
  );
}
```

### Success confirmation pattern
After any important action — purchase, follow, post, save:
```tsx
async function handlePurchase() {
  await completePurchase();

  // 1. Haptic success
  ReactNativeHaptics.notification('success');

  // 2. Visual confirmation (brief, not modal)
  showToast({ type: 'success', message: 'Purchase complete!' });

  // 3. Animate the changed state (not just update text)
  celebrationAnimation.value = withSequence(
    withSpring(1.2),
    withSpring(1)
  );
}
```

### Number counting animations (for stats, scores, counters)
```tsx
import { useSharedValue, useAnimatedProps, withTiming } from 'react-native-reanimated';
import Animated from 'react-native-reanimated';

const AnimatedText = Animated.createAnimatedComponent(Text);

function AnimatedCounter({ value }: { value: number }) {
  const animatedValue = useSharedValue(0);

  useEffect(() => {
    animatedValue.value = withTiming(value, { duration: 800 });
  }, [value]);

  const animatedProps = useAnimatedProps(() => ({
    text: Math.floor(animatedValue.value).toString(),
  }));

  return <AnimatedText animatedProps={animatedProps} />;
}
```

---

## GESTURE PSYCHOLOGY — WHAT FEELS NATURAL

| Gesture | Expected behavior | Never do |
|---|---|---|
| Swipe left on list item | Reveal delete/actions | Open a new screen |
| Long press | Context menu | Nothing (feels broken) |
| Pull down on modal | Dismiss modal | Refresh (confusing) |
| Pinch | Zoom | Nothing (feels broken) |
| Double tap | Like/favorite | Nothing (users expect this) |
| Swipe back (iOS) | Navigate back | Block it |
| Swipe from left (Android) | Navigate back | Block it |

```tsx
// Swipe to delete pattern
const swipeGesture = Gesture.Pan()
  .activeOffsetX([-10, 10])
  .onUpdate((e) => {
    if (e.translationX < 0) {
      translateX.value = Math.max(e.translationX, -80);  // reveal 80pt delete button
    }
  })
  .onEnd((e) => {
    if (e.translationX < -60) {
      translateX.value = withSpring(-80);  // snap open
    } else {
      translateX.value = withSpring(0);   // snap closed
    }
  });
```

---

## TYPOGRAPHY PSYCHOLOGY

### Type scale that communicates hierarchy instantly
```js
// tailwind.config.js — consistent type scale
theme: {
  extend: {
    fontSize: {
      'display': ['34px', { lineHeight: '41px', fontWeight: '700' }],  // Hero titles
      'title1':  ['28px', { lineHeight: '34px', fontWeight: '700' }],  // Screen titles
      'title2':  ['22px', { lineHeight: '28px', fontWeight: '600' }],  // Section headers
      'title3':  ['20px', { lineHeight: '25px', fontWeight: '600' }],  // Card titles
      'body':    ['17px', { lineHeight: '22px', fontWeight: '400' }],  // Body text
      'callout': ['16px', { lineHeight: '21px', fontWeight: '400' }],  // Supporting text
      'caption': ['12px', { lineHeight: '16px', fontWeight: '400' }],  // Labels, metadata
    }
  }
}
```

**Rules:**
- Never use more than 3 font sizes on one screen
- Body text minimum 16pt (17pt ideal for iOS)
- Line height 1.4-1.6× font size for body text
- Don't use font weight for hierarchy alone — combine with size

---

## COLOR PSYCHOLOGY

```tsx
// Semantic color tokens — color communicates meaning
const colors = {
  // Actions
  primary: '#0ea5e9',      // Main CTA — one per app
  destructive: '#ef4444',  // Delete, remove, irreversible
  success: '#22c55e',      // Confirmation, complete
  warning: '#f59e0b',      // Caution, review needed

  // Content
  textPrimary: '#0f172a',   // Main content
  textSecondary: '#64748b', // Supporting, metadata
  textDisabled: '#cbd5e1',  // Inactive states

  // Backgrounds
  bgPrimary: '#ffffff',
  bgSecondary: '#f8fafc',   // Cards, inputs
  bgElevated: '#f1f5f9',    // Modals, sheets
};
```

**Rules:**
- One primary action color per app — used only for the main CTA
- Red = destructive only (not for branding, not for errors that aren't destructive)
- Disabled states should look disabled — 40% opacity or desaturated
- Text on colored backgrounds: check contrast ratio (min 4.5:1 for body, 3:1 for large)
