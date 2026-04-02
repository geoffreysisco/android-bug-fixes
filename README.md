# Android Bug Fix Case Studies

## 1. Duplicate Files + Stale UI in File Manager

### Problem
- Duplicate files appeared in trash through repeated operations
- Deleted files remained visible until manual refresh
- Newly created files were not immediately visible in some views
Verified via automated repro harness across repeated cold starts
### Root Cause
- Inconsistent reload logic across drawer views
- Missing force reload conditions after file operations

### Fix
- Unified reload behavior using a shared `shouldForceReload` condition
- Applied fix across Documents, Recent Files, and APK views

### Result
- No duplicate files
- UI reflects actual filesystem state
- Consistent behavior across all views

### Reference
- Pull Request: https://github.com/TeamAmaze/AmazeFileManager/pull/4589

---

## 2. Sleep Timer Crash Caused by Main Thread DB Access

### Problem
- App crashed when opening the Sleep Timer dialog in debug builds
- Crash message: "I/O on main thread"
- Dialog attempted to read playback timing during UI refresh

### Root Cause
- `SleepTimerDialog.refreshUiState()` depends on playback timing data:
  - `getDuration()`
  - `getPosition()`

- At dialog startup:
  - `currentMedia` is loaded asynchronously via `Single.fromCallable(...)` (IO thread)
  - `refreshUiState()` may run before this completes

- When `currentMedia` is not yet available:
  - Logic falls back to playback/controller methods
  - These methods can trigger `getMedia()`
  - `getMedia()` performs a database read

- This results in **unexpected disk I/O on the main thread**

> Key issue: a race between async media loading and synchronous UI access caused a fallback path that performed hidden database I/O

### Fix
- Removed lazy DB access from:
  - `getDuration()`
  - `getPosition()`
- Updated logic to return:
  - playback service values (if available)
  - cached media values (if already loaded)
  - `Playable.INVALID_TIME` otherwise

### Result
- No crash when opening Sleep Timer dialog
- Playback still pauses correctly when timer expires
- Eliminated main-thread database access in playback getters

### Reference
- Pull Request: https://github.com/AntennaPod/AntennaPod/pull/8366

---

## 3. Out-of-Range Pagination Causing False Network Failure

### Problem
- App occasionally launched into a false network failure state on initial load
- No crash; UI showed failure despite successful API responses
- Issue was non-deterministic and difficult to reproduce

### Root Cause
- Discover mode selected a random start page using a fixed upper bound
- API responses returned a lower `totalPages` value than the randomly selected page
- When the random page exceeded `totalPages`:
  - API returned an empty result set (200 OK)
  - App treated it as a valid empty response
  - Triggered "empty final" state, which surfaced as a network failure

### Fix
- Added guard after receiving `totalPages`:
  - Detect when `browseCurrentPage > browseTotalPages`
  - Re-select a valid random page within bounds
  - Retry request before entering empty-state logic

```java
if (browseMode == BrowseMode.DISCOVER_RANDOM
        && browseTotalPages > 0
        && browseCurrentPage > browseTotalPages) {

    int maxPage = Math.min(browseTotalPages, DISCOVER_RANDOM_START_PAGE_MAX);

    browseCurrentPage = 1 + random.nextInt(maxPage);
    loadNextPageBrowse(cb);
    return;
}
```

### Result
- Eliminated false network failure on startup
- App reliably recovers from out-of-range pagination
- Verified via automated repro harness across repeated cold starts

### Notes
- Built a shell script to repeatedly reinstall, launch, and capture logs
- Confirmed fix by observing:
  - out-of-range page detection
  - successful retry
  - valid data publication to UI

### Reference
- Repository: https://github.com/geoffreysisco/FilmAtlas
- Fix commit: https://github.com/geoffreysisco/FilmAtlas/commit/426f086c8f75865b9c26ac8368a8e73002b86b44
