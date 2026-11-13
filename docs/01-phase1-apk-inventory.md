# Phase 1 — APK Inventory

Target: `com.minimaa.minimal.detox.launcher_1.2.11_arm64-v8a_built_20260725-190436-827.apk`

## Identity (verified)

| Property | Value |
|---|---|
| Package | `com.minimaa.minimal.detox.launcher` |
| Version | 1.2.11 (versionCode 42) |
| ABIs | `arm64-v8a` only |
| Framework | Flutter v2 (legacy split AOT snapshot layout) |
| APK SHA-256 | `bb7a3a85e949e5cb253220c005cf7c7130f114ceafc7afc5c0ebe17240e9ad69` |
| Signing schemes | v1 + v2 + v3 |
| Signing cert SHA-256 | `6e31ba07…` (truncated as recorded in the analysis session) |
| DEX size | ~12,000 code units |

## Native libraries

- `lib/arm64-v8a/libapp.so` — Dart AOT snapshot (analyzed in Phase 2A;
  SHA-256 `a2887b377893ec5706a2af2960a6d88f04ecbdc8adb537306b28e194311a1141`).
- `lib/arm64-v8a/libflutter.so` — Flutter engine.
- `lib/arm64-v8a/libdartjni.so` — JNI bridge (analyzed in Phase 2B(a);
  SHA-256 `5702f3a0…`, 124,744 bytes, 312 exports).

## Declared components (verified via manifest/JADX inventory)

MainActivity, NotificationListener (notification access), DetoxAccessibilityService,
ReminderReceiver (scheduled reminders), ConfirmPinShortcutActivity,
MotionGestureManager, JniFlutterPlugin, plus pairip licensing/integrity classes
and a set of obfuscated classes (`a, c, d, e, f, g, h, j, k, l, m, n, o, q, u`)
that carry the platform-channel adapter logic.

## Tooling and provenance

- REA framework with the caller-supplied JADX headless provider
  (`jadx-headless-mcp-0.7.1-all.jar`, SHA-256
  `6e5eacf500b64292bfb73c49797c1958f6ee44646e43e868039ae7feb573ff75`).
- Extraction was limited to `/tmp/opencode/` workspaces; each extracted artifact
  was digest-verified before analysis.
