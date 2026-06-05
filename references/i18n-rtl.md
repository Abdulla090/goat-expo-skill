# i18n & RTL — Internationalization, Kurdish Sorani, Arabic, Bidirectional Text

## THE RULE

RTL is not "flip everything." It's a complete mirror of the UX.
Text alignment, icons, navigation arrows, swipe directions, layout order —
everything that has directionality must be considered.

Kurdish Sorani (کوردی سۆرانی) is RTL. Arabic is RTL. Hebrew is RTL.
If your app targets any of these audiences, this file is mandatory.

---

## SETUP — expo-localization + i18n-js

```bash
npx expo install expo-localization
npm install i18n-js
```

```ts
// lib/i18n.ts
import { I18n } from 'i18n-js';
import * as Localization from 'expo-localization';

const translations = {
  en: require('../locales/en.json'),
  ku: require('../locales/ku.json'),  // Kurdish Sorani
  ar: require('../locales/ar.json'),  // Arabic
};

export const i18n = new I18n(translations);

// Set locale from device
i18n.locale = Localization.getLocales()[0].languageCode ?? 'en';

// Fall back to English for missing keys
i18n.enableFallback = true;
i18n.defaultLocale = 'en';

// Usage
export const t = (key: string, options?: object) => i18n.t(key, options);
```

```json
// locales/en.json
{
  "common": {
    "save": "Save",
    "cancel": "Cancel",
    "loading": "Loading...",
    "error": "Something went wrong",
    "retry": "Try again"
  },
  "auth": {
    "login": "Log In",
    "register": "Create Account",
    "email": "Email",
    "password": "Password",
    "forgot_password": "Forgot password?"
  },
  "profile": {
    "title": "Profile",
    "edit": "Edit Profile",
    "followers": "%{count} followers",
    "following": "%{count} following"
  }
}
```

```json
// locales/ku.json — Kurdish Sorani
{
  "common": {
    "save": "پاشەکەوت بکە",
    "cancel": "هەڵوەشاندنەوە",
    "loading": "چاوەڕوانبە...",
    "error": "هەڵەیەک ڕوویدا",
    "retry": "دووبارە هەوڵبدەرەوە"
  },
  "auth": {
    "login": "چوونەژوورەوە",
    "register": "هەژمار دروستبکە",
    "email": "ئیمەیڵ",
    "password": "وشەی نهێنی",
    "forgot_password": "وشەی نهێنیت بیرچوویەتەوە؟"
  },
  "profile": {
    "title": "پرۆفایل",
    "edit": "دەستکاریکردنی پرۆفایل",
    "followers": "%{count} شوێنکەوتوو",
    "following": "%{count} شوێندەکەوت"
  }
}
```

---

## ENABLING RTL IN REACT NATIVE

```tsx
// app/_layout.tsx — set RTL BEFORE any component renders
import { I18nManager } from 'react-native';
import * as Localization from 'expo-localization';
import * as Updates from 'expo-updates';

function configureRTL() {
  const locale = Localization.getLocales()[0];
  const isRTL = locale.textDirection === 'rtl';

  // Only force a reload if RTL state needs to change
  if (I18nManager.isRTL !== isRTL) {
    I18nManager.allowRTL(true);
    I18nManager.forceRTL(isRTL);

    // App must reload for RTL to take effect
    // In production: this happens once on first launch with RTL locale
    if (!__DEV__) {
      Updates.reloadAsync();
    }
  }
}

// Call this ONCE at app startup, before render
configureRTL();
```

### Allow user to switch language manually

```tsx
// stores/locale.store.ts
import { create } from 'zustand';
import { persist, createJSONStorage } from 'zustand/middleware';
import { I18nManager } from 'react-native';
import * as Updates from 'expo-updates';
import { i18n } from '../lib/i18n';
import { storage } from '../lib/storage';

const RTL_LOCALES = ['ku', 'ar', 'he', 'fa'];

interface LocaleStore {
  locale: string;
  isRTL: boolean;
  setLocale: (locale: string) => Promise<void>;
}

export const useLocaleStore = create<LocaleStore>()(
  persist(
    (set) => ({
      locale: 'en',
      isRTL: false,
      setLocale: async (locale: string) => {
        const isRTL = RTL_LOCALES.includes(locale);
        i18n.locale = locale;
        set({ locale, isRTL });

        // RTL change requires app reload
        if (I18nManager.isRTL !== isRTL) {
          I18nManager.forceRTL(isRTL);
          await Updates.reloadAsync();
        }
      },
    }),
    {
      name: 'locale',
      storage: createJSONStorage(() => ({
        getItem: (key) => storage.getString(key) ?? null,
        setItem: (key, value) => storage.set(key, value),
        removeItem: (key) => storage.delete(key),
      })),
    }
  )
);
```

---

## RTL LAYOUT — WHAT AUTOMATICALLY FLIPS

When `I18nManager.isRTL = true`, React Native automatically mirrors:

```
✅ Automatically RTL:
- flexDirection: 'row'         → becomes right-to-left
- text alignment (default)     → right-aligned
- padding/margin Start/End     → swapped
- absolute positioning left/right → swapped

❌ Does NOT automatically RTL:
- Icons that have directionality (back arrow, forward arrow)
- Custom animations with explicit translateX
- Images with text baked in
- border-radius (individual corners)
- Absolute positioned elements that you manually placed
```

---

## WRITING RTL-SAFE STYLES

### Use Start/End instead of Left/Right

```tsx
// ❌ Not RTL-safe — hardcoded to one direction
<View style={{ paddingLeft: 16, marginRight: 8 }}>
<Text style={{ textAlign: 'left' }}>

// ✅ RTL-safe — mirrors automatically
<View style={{ paddingStart: 16, marginEnd: 8 }}>
<Text style={{ textAlign: 'auto' }}>  // 'auto' = right in RTL, left in LTR
```

### NativeWind RTL-safe utilities

```tsx
// ✅ NativeWind has RTL-aware utilities
<View className="ps-4 pe-4 ms-2 me-2">   // ps = paddingStart, pe = paddingEnd
<Text className="text-start">            // start-aligned (right in RTL, left in LTR)
```

### RTL-aware flex layouts

```tsx
// Row layout — automatically mirrors in RTL
<View style={{ flexDirection: 'row' }}>
  <Icon />     // appears on right in RTL, left in LTR ✅
  <Text />
</View>

// Explicit RTL control when needed
<View style={{
  flexDirection: I18nManager.isRTL ? 'row-reverse' : 'row'
}}>
```

---

## DIRECTIONAL ICONS — THE MOST COMMON RTL MISTAKE

Icons that show direction must be flipped in RTL:

```tsx
import { I18nManager } from 'react-native';

// Back arrow — points left in LTR, right in RTL
function BackButton() {
  return (
    <Pressable onPress={() => router.back()}>
      <ChevronLeft
        size={24}
        style={{
          transform: [{ scaleX: I18nManager.isRTL ? -1 : 1 }]
        }}
      />
    </Pressable>
  );
}

// Icons that need flipping in RTL:
// - Back/forward arrows (ChevronLeft, ChevronRight, ArrowLeft, ArrowRight)
// - Send button (paper plane pointing right)
// - Progress indicators with directionality
// - Carousel prev/next buttons

// Icons that do NOT need flipping in RTL:
// - Up/down arrows
// - Close/X button
// - Search icon
// - Heart, star, bookmark
// - Play/pause (media controls stay LTR universally)
```

```tsx
// Utility hook
function useDirectionalStyle() {
  return {
    flip: I18nManager.isRTL ? { transform: [{ scaleX: -1 }] } : {},
    textAlign: I18nManager.isRTL ? 'right' : 'left',
    flexStart: I18nManager.isRTL ? 'flex-end' : 'flex-start',
  };
}
```

---

## ARABIC & KURDISH SORANI TEXT RENDERING

### Font considerations

System fonts handle Arabic/Kurdish Sorani well on both platforms:
- iOS: San Francisco (Latin) + system Arabic font — handles mixed text automatically
- Android: Roboto (Latin) + Noto Naskh Arabic — handles mixed text

For custom Arabic/Kurdish fonts:

```tsx
// Recommended Arabic/Kurdish fonts
// - Readex Pro — excellent for Kurdish Sorani + Arabic
// - Cairo — clean, modern Arabic
// - Noto Sans Arabic — Google's universal Arabic font

import { useFonts } from 'expo-font';

const [fontsLoaded] = useFonts({
  'ReadexPro-Regular': require('../assets/fonts/ReadexPro-Regular.ttf'),
  'ReadexPro-Bold': require('../assets/fonts/ReadexPro-Bold.ttf'),
});

// Apply globally in NativeWind
// tailwind.config.js
theme: {
  extend: {
    fontFamily: {
      arabic: ['ReadexPro-Regular'],
      'arabic-bold': ['ReadexPro-Bold'],
    }
  }
}

// Usage
<Text className="font-arabic text-lg">کوردی سۆرانی</Text>
```

### Mixed LTR/RTL text (English inside Kurdish sentence)

```tsx
// React Native handles bidirectional text automatically
// But writingDirection helps disambiguate edge cases
<Text style={{ writingDirection: 'rtl' }}>
  ئەم ئەپەی React Native بۆ iOS و Android دروستکراوە
</Text>

// For mixed content: let the system detect
<Text>
  React Native یەکەمین فریمۆرکە بۆ iOS و Android
</Text>
// System uses Unicode Bidirectional Algorithm — usually correct automatically
```

### Number formatting

```tsx
// Numbers in Arabic/Kurdish contexts
// Eastern Arabic numerals: ٠١٢٣٤٥٦٧٨٩
// Kurdish Sorani uses both Western (0-9) and Eastern Arabic numerals

// Use Intl.NumberFormat for locale-aware formatting
const formatNumber = (num: number, locale: string) =>
  new Intl.NumberFormat(locale).format(num);

formatNumber(1234567, 'en');  // "1,234,567"
formatNumber(1234567, 'ar');  // "١٬٢٣٤٬٥٦٧"
formatNumber(1234567, 'ku');  // "1,234,567" (Kurdish Sorani uses Western numerals)

// Currency
const formatCurrency = (amount: number, locale: string, currency: string) =>
  new Intl.NumberFormat(locale, { style: 'currency', currency }).format(amount);

formatCurrency(50000, 'en', 'IQD');  // "IQD 50,000.00"
formatCurrency(50000, 'ar', 'IQD');  // "د.ع.‏ ٥٠٬٠٠٠٫٠٠"
```

### Date formatting

```tsx
const formatDate = (date: Date, locale: string) =>
  new Intl.DateTimeFormat(locale, {
    year: 'numeric',
    month: 'long',
    day: 'numeric',
  }).format(date);

formatDate(new Date(), 'en');  // "June 3, 2026"
formatDate(new Date(), 'ar');  // "٣ يونيو ٢٠٢٦"
formatDate(new Date(), 'ku');  // "٣ی حوزەیران ٢٠٢٦"

// Relative time
const formatRelative = (date: Date, locale: string) =>
  new Intl.RelativeTimeFormat(locale, { numeric: 'auto' }).format(-2, 'day');
// en: "2 days ago"  ar: "منذ يومين"
```

---

## RTL NAVIGATION

### expo-router handles RTL automatically

```tsx
// Stack navigation: back swipe goes right-to-left in LTR, left-to-right in RTL
// expo-router respects I18nManager.isRTL automatically

// Back button in custom header
function Header({ title }: { title: string }) {
  const { flip } = useDirectionalStyle();
  const insets = useSafeAreaInsets();

  return (
    <View style={{ paddingTop: insets.top }} className="flex-row items-center px-4 py-3">
      <Pressable onPress={() => router.back()} className="p-2">
        <ChevronLeft size={24} style={flip} />  {/* flip in RTL */}
      </Pressable>
      <Text className="flex-1 text-center text-lg font-bold">{title}</Text>
      <View className="w-10" />  {/* balance the back button */}
    </View>
  );
}
```

---

## LANGUAGE SELECTOR UI

```tsx
const LANGUAGES = [
  { code: 'en', name: 'English', nativeName: 'English', rtl: false },
  { code: 'ku', name: 'Kurdish Sorani', nativeName: 'کوردی سۆرانی', rtl: true },
  { code: 'ar', name: 'Arabic', nativeName: 'العربية', rtl: true },
];

function LanguageSelector() {
  const { locale, setLocale } = useLocaleStore();

  return (
    <View>
      {LANGUAGES.map((lang) => (
        <Pressable
          key={lang.code}
          onPress={() => setLocale(lang.code)}
          className="flex-row items-center justify-between p-4 border-b border-gray-100"
        >
          <View>
            <Text className="text-base font-medium">{lang.nativeName}</Text>
            <Text className="text-sm text-gray-500">{lang.name}</Text>
          </View>
          {locale === lang.code && (
            <SymbolView name="checkmark.circle.fill" tintColor="#0ea5e9" size={24} />
          )}
        </Pressable>
      ))}
    </View>
  );
}
```

---

## RTL CHECKLIST

```
Setup:
- [ ] expo-localization installed
- [ ] i18n-js configured with fallback to English
- [ ] Translation files for each language
- [ ] I18nManager.forceRTL called at app startup
- [ ] Locale persisted in MMKV (user preference survives restart)

Styles:
- [ ] paddingLeft/Right → paddingStart/End
- [ ] marginLeft/Right → marginStart/End
- [ ] textAlign: 'left' → textAlign: 'auto'
- [ ] NativeWind: ps-/pe- instead of pl-/pr-

Icons:
- [ ] Back/forward arrows flipped in RTL (scaleX: -1)
- [ ] Send button (paper plane) flipped in RTL
- [ ] Carousel prev/next buttons correct per direction

Text:
- [ ] Arabic/Kurdish font loaded (Readex Pro recommended)
- [ ] Numbers formatted with Intl.NumberFormat
- [ ] Dates formatted with Intl.DateTimeFormat
- [ ] writingDirection set on RTL text containers

Testing:
- [ ] Tested on device with Arabic locale (Settings → Language → Arabic)
- [ ] Tested on device with Kurdish locale (if available)
- [ ] All screens visually verified in RTL mode
- [ ] No text truncation issues in RTL (Arabic text can be longer)
- [ ] Navigation back gesture works correctly in RTL
```
