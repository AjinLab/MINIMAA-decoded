# Phase 2C — Contract Reconciliation and Screen-Time Semantics

Two tasks of Phase 2C: (1) closing the cross-language contract gaps, and
(2) the UsageStats screen-time model. Evidence IDs are REA method inspections
unless noted.

## 1. Dart-side reconciliation (complete)

### Privacy channel — negative finding closed

Complete printable-run scan of the digest-verified `libapp.so`
(SHA-256 `a2887b37…1141`, matching Phase 2A identity; evidence `ev_861d4651…`):
the only privacy-like literals are UI copy and the pickerry.com privacy-policy
URL. No `privacy` channel exists. The nine channel names match the JVM side 1:1.

### Reminder timestamp contract

- **JVM contract (verified).** `MainActivity.scheduleAlarm`
  (`ev_802d7ed4…`) passes `triggerAt` directly to
  `setExactAndAllowWhileIdle` with alarm type **0 = RTC_WAKEUP** (epoch-based).
  The REMINDER handler validates `triggerAt > now` against the epoch clock
  (Phase 2B(c) evidence). `cancelAlarm` (`ev_a39f6821…`) is symmetric.
- **Dart call site (inference, high confidence).** Dart AOT call-site
  disassembly is not possible with the available tooling; epoch-millis is
  inferred from the JVM contract, corroborated by snapshot tokens
  (`millisecondsSinceEpoch`, `TZDateTime.fromMillisecondsSinceEpoch`).
- **Outcome.** The Phase 2B(c) "reminders never fire" defect claim is
  **retracted**; the pipeline is coherent.

## 2. Screen-time semantics

### Data source and day boundary (verified, `ev_64311751…`)

- `UsageStatsManager` via `getSystemService("usagestats")`.
- Day start = **local midnight** (device-timezone `Calendar` with
  hour/minute/second/millisecond zeroed; DST-normalized). Window =
  `[local midnight, now]`.
- `queryEvents` is called **twice** over the same window (two independent
  cursors; one for a prefill pass, one for the main loop).
- **No usage-access permission check** in any handler: if the special
  `PACKAGE_USAGE_STATS` access is not granted, the platform returns no events
  and all usage methods silently report 0 (platform-behavior note).

### Core algorithm `computeAllAppUsageTodayMs` (verified)

Event constants verified against AOSP `UsageEvents.java`
(android.googlesource.com): ACTIVITY_RESUMED=1 (alias MOVE_TO_FOREGROUND),
ACTIVITY_PAUSED=2 (alias MOVE_TO_BACKGROUND), SCREEN_INTERACTIVE=15,
SCREEN_NON_INTERACTIVE=16, ACTIVITY_STOPPED=23.

1. **Midnight-span prefill (one-shot):** reads only the *first* event of the
   second cursor; if it is PAUSED or SCREEN_NON_INTERACTIVE with a package,
   credits that package `[midnight → first-event ts]` — capturing a session
   spanning midnight. Screen events usually carry no package, so the 16 branch
   rarely fires in practice (platform-dependent).
2. **Main-loop state machine** (tracking flag + tracked package + last-event
   timestamp, initialized to midnight; timestamp advances on every
   package-bearing event):
   - RESUMED(1): credit the previously tracked (different) app
     `(resumeTs − lastTs)`, then track the new package. Same-app re-resume
     continues the session without crediting.
   - PAUSED(2): if the tracked package matches, credit `(pauseTs − lastTs)`.
     Tracking state is *not* cleared (jadx renders the accumulation
     out-of-line — a linearization artifact of an out-of-loop branch target;
     condition semantics attributed with medium-high confidence).
   - SCREEN_INTERACTIVE(15): sets the tracking flag only.
   - SCREEN_NON_INTERACTIVE(16): credits the open tracked package and clears
     tracking — **screen-off ends the current session**.
   - Post-loop: still-tracked package is credited `(now − lastTs)`.
   - Non-positive entries are dropped.

### Derived quirks (analysis; medium confidence)

- **Pause does not close a session**: the gap between a pause and the next
  app's resume is credited to the previously tracked app (home-screen dwell
  time inflates the app used before it).
- **Undercount risk**: the last-event timestamp advances on *all*
  package-bearing events (including non-boundary types such as
  USER_INTERACTION), so intra-session time before such events is skipped at
  the next credit. Impact depends on which event types the device build emits.
- Two identical `queryEvents` calls double platform query cost (minor).

### Aggregation wrappers (all verified)

- `computeTotalScreenTimeTodayMs` (`ev_9738f82b…`): sums the map over all
  packages **except the app's own package** (self-exclusion).
- `computeAppUsageTodayMs(pkg)` (`ev_f17c2bfe…`): map lookup, default 0.
- Execution: coroutine workers `$9$7`–`$9$10` run on a lifecycle scope + IO
  dispatcher, posting results via the main `Handler`; exceptions become channel
  errors (`ev_2ffafb3c…`, `ev_3e9b0f59…`, `ev_b2a4ee39…`, `ev_15bd8e0b…`).

### `getAllLastUsedTimestamps` (verified, `ev_15bd8e0b…`)

Not event-based: `queryUsageStats(INTERVAL_DAILY, now − 14 days, now)`, taking
the **max `LastTimeUsed`** per package across daily buckets (positive values
only) → `Map<pkg, epoch-ms>`. Apps idle > 14 days are absent; the value is the
platform's approximate `LastTimeUsed` field, not session-derived.

### Foreground detection — two distinct paths (verified)

- **`getForegroundApp` channel** (`ev_b4d41159…`, `ev_32c44c8d…`,
  `ev_d9da6118…`): single-flight guard (atomic compare-and-set; busy →
  `success(null)`), then a worker thread runs a **1-hour** `queryEvents` scan
  returning the last RESUMED package; guard cleared in `finally`. (A doubled
  `== 1` comparison in the decompiled text is a jadx artifact of equivalent
  event-type aliases.)
- **`getForegroundPackageName` private method** (`ev_273d2ca8…`):
  accessibility-first (`DetoxAccessibilityService.getCurrentForegroundPackage`,
  rejected if null/self/`com.android.systemui`; an `isPlaying` clause in the
  acceptance condition is dead logic that cannot affect the outcome), with a
  **15-minute UsageStats fallback** (last resume without a matching
  PAUSED/STOPPED=23). On mismatch it **prefers the UsageStats result**
  (logged under tag `DETOX_FG`). Its caller trace is queued for Phase 2C
  task 3.

## 3. ReminderReceiver (verified, `ev_bd1ce816…`)

Contains exactly `<init>`, `onReceive`, `showNotification` — alarm scheduling
lives in `MainActivity` (`scheduleAlarm`/`cancelAlarm`), not the receiver.
