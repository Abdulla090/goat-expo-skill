# expo-file-system — SDK 56 API Shifts (June 2026)

Breaking / behavioral changes when upgrading from SDK 55. Changelog: https://expo.dev/changelog/sdk-56

---

## ASYNC BY DEFAULT

| Operation | SDK 56 behavior |
|-----------|-----------------|
| `copy()` / `move()` | **Async** — `await` required |
| Sync immediate need | `copySync()` / `moveSync()` |
| Large downloads | Progress callbacks + **AbortSignal** cancel |
| Uploads | `createUploadTask()` task-based API |
| Multi pick | Multiple files + MIME filters in one call |

**Agent rule:** Do not `copy()` without `await` after SDK 56 migration.

---

## MIGRATION SNIPPET

```tsx
import * as FileSystem from 'expo-file-system';

// Async (default)
await FileSystem.copyAsync({ from: src, to: dest });

// Sync only when you truly need blocking semantics on same tick
FileSystem.copySync(src, dest);

// Download with cancel
const controller = new AbortController();
const task = FileSystem.createDownloadResumable(url, dest, {}, onProgress);
// task.cancelAsync() or AbortSignal per SDK docs
```

---

## COMMON BUGS AFTER UPGRADE

- Fire-and-forget `copy()` → race / missing files on next screen
- Assuming sync MD5/hash on huge files without streaming — check docs for large-file fixes in SDK 56
- iOS disk space checks — use updated APIs for free space reporting

Pair with TanStack Query mutations for upload progress UI; never block UI thread with sync IO on main path.
