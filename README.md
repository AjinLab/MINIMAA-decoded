# MINIMAA-decoded — Reverse-Engineering Findings

Static-analysis notes for the Android APK
`com.minimaa.minimal.detox.launcher_1.2.11_arm64-v8a_built_20260725-190436-827.apk`
(SHA-256 `bb7a3a85e949e5cb253220c005cf7c7130f114ceafc7afc5c0ebe17240e9ad69`).

Analysis was performed **read-only** with the REA reverse-engineering framework
(JADX headless provider for DEX/JVM, structural parser for Dart AOT snapshots).
The APK was never executed or modified; all extraction went to `/tmp/opencode/`
workspaces, and the original file's digest was re-verified after each phase.

## Conventions

- **Verified** — backed by a REA evidence ID (`ev_…`) at source/bytecode level.
- **Derived** — logical consequence of verified facts; flagged where applicable.
- **Hypothesis / unresolved** — explicitly labeled; not asserted as fact.
- Implementation logic is **paraphrased**, not reproduced. Evidence IDs refer to
  REA analysis sessions against the digest-verified APK above.

## Document index

| Doc | Phase | Scope |
|-----|-------|-------|
| [01](docs/01-phase1-apk-inventory.md) | Phase 1 | APK inventory, identity, signing |
| [02](docs/02-phase2a-dart-aot.md) | Phase 2A | Dart AOT snapshot structure, channel literals, privacy negative finding |
| [03](docs/03-phase2b-a-jni-bridge.md) | Phase 2B(a) | `libdartjni.so` JNI bridge — jnigen runtime only |
| [04](docs/04-phase2b-b-dex-jvm-implementation.md) | Phase 2B(b) | DEX/JVM implementation inventory, key classes, storage keys |
| [05](docs/05-phase2b-c-channels-and-enforcement.md) | Phase 2B(c) | Channel dispatch table, outbound pushes, grace-enforcement loop, integrity |
| [06](docs/06-phase2c-contracts-and-screen-time.md) | Phase 2C | Dart/JVM contract reconciliation, reminder-alarm retraction, screen-time semantics |

## Headline findings

1. **All app semantics are Kotlin/JVM.** Dart (Flutter) supplies UI; every
   behavior contract crosses 9 platform channels (8 `MethodChannel` +
   1 `EventChannel`). The JNI library is a generic jnigen runtime, not app logic.
2. **Channel set reconciled 1:1 across languages**; no `privacy` channel exists
   (earlier hypothesis retired; the only privacy literal is a policy URL).
3. **Reminder scheduling is coherent**: epoch-millis validation on the JVM side
   plus `RTC_WAKEUP` alarm type. A Phase 2B(c) "reminders never fire" defect
   claim was **retracted** in Phase 2C after re-verifying the alarm-type constant.
4. **Screen-time accounting** is UsageStats event-stream based with a local
   midnight boundary, a midnight-span prefill heuristic, and screen-on/off
   awareness — with documented over/under-count quirks.

## Status

- Phases 1, 2A, 2B(a), 2B(b), 2B(c), and 2C tasks 1–2 (contract reconciliation,
  screen-time semantics) are complete and documented.
- **In progress (not yet documented here):** Phase 2C task 3 — enforcement
  controls (`setBackLock`, `minimizeApp`, `lockScreen`, `syncMotionGestures`),
  `getForegroundPackageName()` call graph, and `MotionGestureManager` internals;
  then pairip licensing flow (task 4).
