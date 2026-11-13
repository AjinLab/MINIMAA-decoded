# Phase 2B(c) — Channel Dispatch and Enforcement Architecture

Evidence: REA method/class inspections of the digest-verified APK; dispatch
regions recovered from `MainActivity.configureFlutterEngine$lambda$24` and the
adapter handler classes.

## 1. Channel dispatch table (verified)

Nine channels: 8 `MethodChannel` + 1 `EventChannel`. Handler classes:
`k` (APP_LAUNCH / MEDIA / DETOX), `l` (WALLPAPER / METADATA),
`m` (NOTIFICATION / REMINDER); supporting adapter classes `c, e, g, h, a, d,
L0.r, n, o`. Channel objects: `detoxMethodChannel`, `intentChannel`,
`metadataChannel`, `notificationChannel`, `reminderChannel`, plus
`mediaEventSink` (EventSink).

### Inbound method sets (verified per dispatch region)

| Channel | Methods |
|---|---|
| APP_LAUNCH | launchApp, getAppShortcuts, launchShortcut |
| MEDIA | dispatchMediaAction, play, pause, next, previous, toggle |
| DETOX | 33 methods (incl. usage/foreground queries — see Phase 2C doc) |
| WALLPAPER | getWallpaperBytes (via WallpaperManager → JPEG quality 85) |
| METADATA | getInitialState |
| NOTIFICATION | dismissNotifications, cancelNotification, dismissNotification, getInitialNotifications, checkPermission, openSettings, openNotification |
| REMINDER | checkExactAlarmPermission, cancelReminder, openExactAlarmSettings, scheduleReminder |

### Outbound pushes (verified)

- Detox events: `onAppDetected`, `onUrlDetected`, `onTimerExpired`,
  `onPackageChanged`; lifecycle events
  installed/updated/changed/profile_changed, debounced.
- Notifications: `posted` / `removed`; while the Flutter engine is unattached,
  events queue in `pendingNotifications` and flush later.
- Metadata: `update` / `updateState`.
- Media: `mediaEventSink.success(map)` on state changes.
- Intents: `onHomeIntent`, `onPomodoroIntent`.

## 2. Grace-enforcement loop (citation-complete, verified chain)

`grantTemporaryAccess → grantGrace → scheduleEnforcementTimer → e(1) →
lambda$49 → lambda$47` (violation check) `→ performGlobalAction(2)`
(GLOBAL_ACTION_HOME) `→ MainActivity → e(0) → lambda$46 → onTimerExpired`.
On reboot, `rescheduleEnforcementTimers` re-arms pending grace timers.

Interpretation (derived): when a blocked app is opened during a detox session,
the user gets a temporary grace window; at expiry a timer re-checks the
foreground package and, on violation, sends the device HOME.

## 3. Integrity surface (verified, `ev_a292ee1f…`)

`verifyIntegrity` returns exactly three checks: `{ signatures, installer,
isDebug }` — signing-cert match, installer-package check, and debug flag.
No deeper anti-tampering logic was found at this surface (pairip classes are
separate; bodies pending Phase 2C task 4).

## 4. Reminder pipeline (verified; one retraction)

- `ReminderReceiver.showNotification` posts to the `task_reminders` channel.
- `MainActivity.showNativeNotification` posts to `pomodoro_notifications` with
  tap action `open_pomodoro`, notification id 1001.
- `MainActivity.scheduleAlarm(id, title, message, triggerAt)`
  (`ev_802d7ed4…`): `AlarmManager.setExactAndAllowWhileIdle(0, triggerAt, …)`.
  Alarm type 0 = **RTC_WAKEUP** (epoch-based wall clock);
  requestCode = `id.hashCode()`; flags = `FLAG_UPDATE_CURRENT | FLAG_IMMUTABLE`.
- `cancelAlarm` (`ev_a39f6821…`) uses the same requestCode derivation →
  symmetric cancellation.

> **Retraction notice.** The original 2B(c) report hypothesized a defect in
> which reminders "never fire" because the epoch trigger was allegedly fed to
> an elapsed-clock alarm. That was based on an inverted reading of the
> AlarmManager type constants. With the correct mapping (0 = RTC_WAKEUP), the
> pipeline is coherent and the claim is **withdrawn** (Phase 2C re-verification).

## 5. Privacy negative finding (verified, complete coverage)

No `privacy` namespace exists JVM-side, matching the Phase 2A/2C Dart-side
negative finding. All privacy-related strings are UI copy or a policy URL.

## 6. Limitations

- `MainActivity.<init>` does not disassemble (Java and smali both fail), so the
  channel-name literals inside it are inferred from dispatch regions, not read
  directly from that method.
- pairip `Application`/`LicenseClientIA` bodies were not yet inspected at this
  phase (queued as Phase 2C task 4).
