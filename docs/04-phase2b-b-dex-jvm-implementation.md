# Phase 2B(b) — DEX/JVM Implementation Inventory

Evidence: REA `inspect_android_class` / `inspect_android_method` across the
digest-verified APK (JADX headless provider).

## App classes analyzed (verified)

| Class | Role (as evidenced by bodies) |
|---|---|
| `MainActivity` | Flutter host; 127-method inventory; owns the DETOX/usage/foreground logic and alarm scheduling |
| `NotificationListener` | Notification access service; relays posted/removed notifications to Dart |
| `DetoxAccessibilityService` | Accessibility-based enforcement; exposes current foreground package |
| `DetoxStateManager` | Detox state store; backs grace/enforcement state |
| `ReminderReceiver` | Broadcast receiver for fired reminder alarms → shows notifications |
| `MediaRepository` | Singleton media-session state (title/artist/package/artwork/position/duration/isPlaying) + listener interface |
| `MotionGestureManager` | Sensor-driven gesture handling (detailed trace pending, Phase 2C task 3) |
| `ConfirmPinShortcutActivity` | Launcher-shortcut entry point |
| `JniFlutterPlugin` | jnigen plumbing plugin |
| pairip classes | Licensing/integrity scaffolding (bodies pending, Phase 2C task 4) |
| Obfuscated adapters | `a, c, d, e, f, g, h, j, k, l, m, n, o, q, u` — MethodChannel/EventChannel handlers and support classes |

## Known analysis gaps (verified negative)

- `MainActivity.<init>` **resists disassembly** (both Java decompile and smali
  fallback fail). Consequently the exact DEX channel-name string literals inside
  the constructor are unverified; channel identity is established instead by the
  dispatch regions (`configureFlutterEngine$lambda$24` etc.) and matched 1:1
  against the Dart snapshot.
- `MainActivity.<clinit>` contains no `const-string` instructions.

## Persistence keys (verified from string pools / bodies)

`detox_prefs`:
`web_detox_enabled`, `porn_block_enabled`, `blocked_packages`,
`blocked_websites`, `custom_porn_domains`, `last_blocked_app`, `grace_<id>`,
`mindful_pause_enabled`.

`FlutterSharedPreferences`:
`flutter.notificationFilterEnabled`, `flutter.filteredPackages`.

## Media pipeline (verified)

`MediaRepository.INSTANCE` is a process-wide singleton carrying
title/artist/package/artwork/position/duration/isPlaying. Updates are pushed to
Dart via the `media_events` `EventChannel` sink (`mediaEventSink.success(map)`).
`MainActivity` implements `MediaRepository.Listener.onMediaStateChanged`.

## Coroutine bridging

Classes named `$9$7` … `$9$13` in `MainActivity` are Kotlin coroutine
suspend-lambda adapters that run work off the main thread and post
`result.success(...)` / `result.error(...)` back through the main `Handler`
(verified for `$9$7`–`$9$10` in Phase 2C; same pattern family throughout).
