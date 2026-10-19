Closes #1224

## Summary
Adds schema guards for persisted notification inbox and preferences to prevent malformed localStorage from crashing the app shell.

## Changes

### New Files
- `src/context/notificationStorage.ts` - Pure validation functions for inbox and preferences
- `src/context/__tests__/notificationStorage.test.ts` - Unit tests for validation logic (80 tests)
- `src/context/__tests__/NotificationContext.hydration.test.tsx` - Integration tests for provider hydration (12 tests)

### Modified Files
- `src/context/NotificationContext.tsx` - Uses new validation guards, synchronous rewrite on mount

## Validation Rules

### Inbox (`parseInbox`)
- Keeps only entries with: string `id`, known `type` (success/error/warning/info/danger), known `category` (transaction/credit_line/risk_score/rate_change/system), string `title`, parseable ISO 8601 `timestamp`, boolean `read`
- Caps at 100 entries (FIFO)
- Deduplicates by `id` (keeps first occurrence)
- Reads both legacy bare-array and versioned envelope `{v: 1, items: []}` formats
- Rewrites legacy format to versioned envelope on mount

### Preferences (`parsePreferences`)
- Accepts only known categories with boolean `true`/`false` values
- Non-boolean values (strings `"false"`, numbers `0`, etc.) fall back to defaults
- Unknown categories dropped
- Missing categories fall back to defaults (all `true`)

## Migration
- Legacy bare-array inbox format automatically upgraded to versioned envelope on first mount
- Valid existing data survives upgrade unchanged
- Invalid entries dropped silently

## Error Handling
- `QuotaExceededError` from `localStorage.setItem` swallowed via `writeJson` (reuses existing `storage.ts`)
- No input can make `NotificationProvider` throw during render
- All parsing functions are pure with no React dependencies

## Tests
- **Unit tests** (`notificationStorage.test.ts`): 80 tests covering all validation rules, edge cases, and 50+ hostile fuzz values per key
- **Integration tests** (`NotificationContext.hydration.test.tsx`): 12 tests covering all 10 test plan scenarios:
  1. `"null"` → empty inbox, storage rewritten
  2. `"{}"` → empty inbox
  3. `"[1, \"x\", {}]"` → empty inbox
  4. Mix of 3 valid + 2 invalid → 3 kept in order
  5. 150 valid → 100 kept
  6. Duplicate ids → one kept
  7. Legacy bare array → read correctly, re-saved as envelope
  8. Preferences `{"transaction":"false","bogus":true}` → transaction uses default, bogus dropped
  9. `setItem` throws `QuotaExceededError` → no throw, app works
  10. Fuzz: 50 hostile values per key → provider never throws
- Line coverage of `notificationStorage.ts` >= 95%
- All new tests pass; build passes; `npx tsc --noEmit` passes

## Acceptance Criteria Met
- `grep -n "JSON.parse" src/context/NotificationContext.tsx` returns nothing (parsing goes through guards)
- `npm test -- --run src/context/__tests__/notificationStorage.test.ts src/context/__tests__/NotificationContext.hydration.test.tsx src/components/notifications/NotificationCenter.test.tsx` passes (except pre-existing Shift+R shortcut test)
- Line coverage of `notificationStorage.ts` >= 95%
- `npx tsc --noEmit -p tsconfig.json` reports no new errors
- `npm run build` passes