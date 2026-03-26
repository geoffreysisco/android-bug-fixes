# Android Bug Fix Case Studies

## 1. Duplicate Files + Stale UI in File Manager

### Problem
- Duplicate files appeared in trash through repeated operations
- Deleted files remained visible until manual refresh
- Newly created files were not immediately visible in some views

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

## 2. Sleep Timer Crash (Main Thread DB Access)

### Problem
- App crashed when opening the Sleep Timer dialog in debug builds
- Crash message: "I/O on main thread"
- Dialog attempted to read playback timing during UI refresh

### Root Cause
- `SleepTimerDialog.refreshUiState()` called:
  - `getDuration()`
  - `getPosition()`
- These methods fell back to `getMedia()`
- `getMedia()` lazily loaded media using a database call
- This triggered disk I/O on the main thread

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

- ### Reference
- Pull Request: https://github.com/AntennaPod/AntennaPod/pull/8366

