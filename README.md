# Android Bug Fix Case Studies

## Duplicate Files + Stale UI in File Manager

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


