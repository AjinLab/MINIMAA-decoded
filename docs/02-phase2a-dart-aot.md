# Phase 2A — Dart AOT Snapshot

Evidence: `ev_861d4651…` (complete structural inspection, arm64-v8a).

## Snapshot structure (verified)

`libapp.so` is a **legacy split AOT snapshot** (Dart 2.14-style FullAOT, kind 3)
built with **Dart SDK 3.11.5**.

| Section | Size (bytes) | Notes |
|---|---|---|
| `_kDartVmSnapshotInstructions` | 92,704 | |
| `_kDartIsolateSnapshotInstructions` | 6,006,240 | compiled Dart code |
| `_kDartVmSnapshotData` | 18,384 | kind 3, magic valid |
| `_kDartIsolateSnapshotData` | 4,621,568 | kind 3, magic valid |

Snapshot hash: `78da37fed6bf1489361a312568249f3f`.
Unified `_kDartSnapshotData/_kDartSnapshotText` layouts are unsupported by the
parser; this binary uses the legacy split form, which is fully covered.

## String-pool projection (partial by design)

The structural parser exposes a bounded projection: 500 package URIs,
197 Dart URIs, 2,000 class-like tokens, 33,756 printable runs. Because the
projection is bounded, a **complete printable-run scan** of the extracted,
digest-verified `libapp.so` (62,683 runs) was used in Phase 2C to close
coverage for channel-name reconciliation.

## Dart-side channel literals (verified)

Exactly nine channel names, matching the JVM side one-to-one:

```
com.minimaa.minimal.detox.launcher/app_launch
com.minimaa.minimal.detox.launcher/detox
com.minimaa.minimal.detox.launcher/intents
com.minimaa.minimal.detox.launcher/media_control
com.minimaa.minimal.detox.launcher/media_events
com.minimaa.minimal.detox.launcher/metadata
com.minimaa.minimal.detox.launcher/notifications
com.minimaa.minimal.detox.launcher/reminders
com.minimaa.minimal.detox.launcher/wallpaper
```

## Dart-side identifiers relevant to later phases (verified)

- Reminders: `TaskReminderManager`, `task_reminder_manager`, `scheduleReminder`,
  `cancelReminder`, `preciseRemindersEnabled`, `taskReminderLeadMinutes`,
  `setTaskReminderLeadMinutes`, `tasksEnableReminders`, `_updateReminder`.
- Outbound-event callback names visible in the snapshot: `onAppDetected`,
  `onUrlDetected`, `onTimerExpired`, `onPackageChanged`, `onHomeIntent`,
  `onPomodoroIntent`.
- Time library tokens: `millisecondsSinceEpoch`,
  `TZDateTime.fromMillisecondsSinceEpoch`, `DateTime.now` (generic library
  tokens — corroboration only, not call-site proof).

## Privacy negative finding (verified, complete coverage)

There is **no `privacy` channel**. An early Phase 2A enumeration produced a
`privacy` token from the identifier pool; the Phase 2C raw scan shows the only
privacy-like literals are UI copy (e.g. "100% Privacy-First Guarantee",
settings strings) and the single URL
`https://apps.pickerry.com/apps/com.minimaa.minimal.detox.launcher/privacy-policy`.
No platform-channel endpoint named `privacy` exists in either language.

## Limitations

- AOT call-site disassembly is not available with this tooling: the *values*
  Dart passes at call sites (e.g. to `scheduleReminder`) cannot be read
  directly, only inferred from the JVM-side contracts.
