# Phase 2B(a) — JNI Bridge (`libdartjni.so`)

Evidence: REA ELF/exports inspection of the digest-verified extracted copy at
`/tmp/opencode/minimaa-phase2b/lib/arm64-v8a/libdartjni.so`.

## Facts (verified)

| Property | Value |
|---|---|
| SHA-256 | `5702f3a0…` (truncated as recorded) |
| Size | 124,744 bytes |
| Exports | 312 |
| `JNI_OnLoad` | **absent** |
| `com/minimaa/*` native symbols | **none** |

## Conclusion

The library is the standard **jnigen / `libdartjni` generic runtime**: reference
management, JNI attachment helpers, and generic Java-object bridging used by
jnigen-generated bindings. It contains **no application-specific native code**:

- No native methods registered for any `com.minimaa` class.
- No app logic in the exported symbol set.

## Architectural implication (derived)

Every app behavior the Dart side needs must be implemented in Kotlin/JVM behind
the platform channels. The presence of `libdartjni.so` (and `JniFlutterPlugin`)
is consistent with jnigen-based Java interop plumbing, **not** with native
enforcement logic. (Coverage note: conclusion rests on the complete 312-export
list plus the absence of package-prefixed natives — complete negative evidence
at the export level; runtime symbol registration is not observable statically.)
