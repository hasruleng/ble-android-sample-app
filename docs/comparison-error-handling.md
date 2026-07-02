# Comparison Report — Error Handling & BLE Error Coverage

**Tapkey's Repos compared:**
- **Sample** — `android-sample-app` (this repo, Java, discontinued, v2.19.2.2)
- **Template** — `tapkey-keyring-app-template-android` (Kotlin, current official reference)

**Study question:** Does the template have meaningfully more complete error/retry coverage for BLE, and should study focus switch there for those patterns?

---

## Summary Verdict

| Area | Sample app | Template | Study source |
|------|-----------|----------|--------------|
| BLE connection lifecycle (scan/stop/cleanup) | Complete | Complete | Either — both identical |
| Command result code handling | **Incomplete** — single TODO | **Complete** — full enum mapping | **→ Template** |
| Exception type differentiation | Generic catch | `ConnectionLostException`, `BleException`, `ServerCommunicationException` mapped | **→ Template** |
| Error severity classification | None | Warning / Error / Log per code | **→ Template** |
| Retry / reconnect logic | **None** | **None** | Neither — pattern absent in both |
| Timeout handling | 15 s hardcoded | 15 s BLE + 60 s NFC | Sample sufficient |
| User cancellation | Manual cancel button | Manual cancel | Either |
| Token refresh on expiry | Redirect to login | Firebase sign-out + redirect | Either |
| Network-state-aware messaging | No | Yes (online/offline distinction) | **→ Template** |
| NFC/HCE as second BLE-adjacent transport | No | Yes | **→ Template** (if NFC is relevant) |

**Decision:** Stay with the sample app as the primary study source for BLE scan/connect/disconnect lifecycle — it is simpler and every step is traceable. Switch to the template specifically for error code and exception handling patterns, where it is significantly more complete.

---

## Retry / Reconnect — Absent in Both

Neither codebase contains retry loops, exponential backoff, or automatic reconnection on BLE drop. Both treat BLE failure as terminal: the exception propagates to the UI layer, a message is shown, and the user must tap again.

This is the most important finding for the parent project (PWA relay design): **the reference implementation for reconnect logic does not exist in either Tapkey codebase.** Any reconnect/idempotency strategy for the ESP32-S3 relay must be designed from scratch.

---

## Command Result Codes — Template is Complete, Sample has a TODO

### Sample app
`KeyListFragment.java:116–123` — switch on `commandResultCode` handles only `Ok`. All other codes fall through to a generic toast:

```java
switch (commandResult.getCommandResultCode()) {
    case Ok:
        return true;
    // TODO: Issue meaningful error messages for different error codes here.
    // Functionality to do this will be provided in future versions of the App SDK.
}
```

### Template
`MessageResolver.kt` maps every `CommandResultCode` to a string resource and a severity level:

| Code | Message key | Severity |
|------|-------------|----------|
| `Ok` | — | Log |
| `WrongLockMode` | `wrong_lock_mode` | Warning |
| `LockVersionTooOld` | `lock_version_too_old_error` | Warning |
| `LockVersionTooYoung` | `lock_version_too_young_error` | Warning |
| `LockNotFullyAssembled` | `lock_not_fully_assembled` | Error |
| `ServerCommunicationError` | online/offline-aware | Error |
| `LockDateTimeInvalid` | `lock_date_time_invalid_error` | Warning |
| `TemporarilyUnauthorized` | `unauthorized_not_yet_valid_error` | Warning |
| `Unauthorized_NotYetValid` | `unauthorized_not_yet_valid_error` | Warning |
| `Unauthorized` | `unauthorized_error` | Warning |
| `LockCommunicationError` | `lock_communication_error` | Error |
| `UserSpecificError` | `generic_error` | Error |
| `TechnicalError` | `generic_error` | Error |

Severity drives UI: Warning → yellow icon (`ic_open_warning`), Error → red icon (`ic_open_error`).

The template also maps `ValidityError` (a separate enum for TLCP-layer validation failures): `SessionBroken`, `LockProtocolError`, `TransportProtocolError`, `UnexpectedLockResponseError`, `ConcurrencyError`, `DifferentDevice`.

**Study use:** Reference `MessageResolver.kt` to understand the full space of things that can go wrong in a BLE command round-trip. The enum names are protocol-meaningful even if the string values are Tapkey-specific.

---

## Exception Type Differentiation

### Sample app
A single `catchOnUi(e -> { Log.e(...); toast(); return false; })` catches everything.

### Template
`MessageResolver.getMessage(Exception)` unwraps exception types before mapping:

```kotlin
is AsyncException    → unwrap to syncSrcException, recurse
is TkException       → map CommandResult codes (table above)
is ConnectionLostException  → "connection_lost"
is BleException             → "ble_connection_err"
is ServerCommunicationException → "server_communication_error"
is IOException              → "unknown" / generic
```

`ConnectionLostException` and `BleException` are explicitly named — these represent mid-transfer drop and BLE-layer failure respectively. Even though neither triggers reconnect, **their existence as distinct types** is useful for the parent project: it confirms that the Tapkey SDK internally distinguishes "connection was lost" from "BLE adapter error", meaning a relay implementation should treat them differently too.

---

## BLE Connection Lifecycle — Both Equivalent

Both repos implement the same lifecycle:

```
onResume  → startForegroundScan()
           → addObserver(locksChanged)
           → addObserver(keyUpdate)

onPause   → close bleScanObserverRegistration
           → close bleObserverRegistration
           → close keyUpdateObserverRegistration
```

The sample app's `KeyListFragment.java:224–260` is a clean, readable example of this pattern and remains the preferred study source — the template wraps the same logic in Kotlin coroutines which adds ceremony without adding clarity for this specific lifecycle question.

---

## NFC / HCE Transport (Template-only)

The template adds a second transport path via NFC/HCE (`HceTagHandler.kt`, `TapkeyHceService.java`). The same `CommandExecutionFacade.triggerLock` is called regardless of transport, with explicit disconnect handling:

```kotlin
val promise = ConnectionUtils.runAndDisconnectAsync(lock) {
    tapkeyServiceFactory.commandExecutionFacade.triggerLock(tlcpConnection, CancellationTokens.None)
}
```

`runAndDisconnectAsync` is the only place in either repo where explicit post-command disconnect appears. This is relevant for the PWA relay design: it confirms that the pattern is "open → command → explicit close", not "open and leave the SDK to manage connection lifetime."

Not relevant to the parent project's BLE path, but worth noting if NFC relay is ever considered.

---

## Token Refresh on Failure

Both redirect to the login screen on `onRefreshFailed`. The template adds:

1. Firebase Analytics event (`TOKEN_REFRESH_FAILED`) before redirect
2. Explicit `FirebaseAuth.getInstance().signOut()` before redirect

This is an auth-stack detail, not a BLE pattern. No study action required.

---

## Files Referenced

**Sample app:**
- [KeyListFragment.java](../app/src/main/java/net/tpky/demoapp/KeyListFragment.java) — BLE lifecycle, generic command error catch
- [KeyItemAdapter.java](../app/src/main/java/net/tpky/demoapp/KeyItemAdapter.java) — timeout, user cancellation
- [SampleTokenRefreshHandler.java](../app/src/main/java/net/tpky/demoapp/SampleTokenRefreshHandler.java) — onRefreshFailed

**Template:**
- `tapkey-keyring-app-template-android/app/src/main/java/io/tapkey/util/MessageResolver.kt` — full error code + exception type mapping
- `tapkey-keyring-app-template-android/app/src/main/java/io/tapkey/wl/app/ui/keys/KeysListAdapter.kt` — command execution, severity-based UI
- `tapkey-keyring-app-template-android/app/src/main/java/io/tapkey/wl/app/hce/HceTagHandler.kt` — NFC transport, explicit disconnect pattern

---

## Checklist Updates for CLAUDE.md

- [x] Compared error-handling paths in both repos
- **Finding:** Template has meaningfully more complete error/result-code coverage; sample is sufficient for BLE lifecycle and scan/connect flow
- **Action:** Reference `MessageResolver.kt` in the template when documenting error paths; do not switch primary study to template

No retry or reconnect logic exists in either repo — this gap must be addressed in the parent project's own design.
