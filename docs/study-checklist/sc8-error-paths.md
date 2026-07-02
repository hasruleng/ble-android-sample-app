# SC8 — Error Paths

**Question:** What are the failure modes across scan timeout, connect fail, credential reject, and drop mid-transfer?

**Reference:** [comparison-error-handling.md](../comparison-error-handling.md) (broad comparison) and [ref-message-resolver.md](../ref-message-resolver.md) (full error taxonomy from the newer template).

---

## Summary matrix

| Failure | Where caught | User-visible | Retry? | Notes |
|---------|--------------|--------------|--------|-------|
| Scan timeout | `KeyListFragment.onResume` try/catch | Silent (log only) | No | See `KeyListFragment.java:224-232` |
| Permission denied | `requestPermissionLauncher` callback | Snackbar with rationale or Settings link | Manual (user re-grants) | See [UC4](../uc4-ble-permissions.md) |
| Connect fail | `catchOnUi` in `triggerLock` | Toast `trigger_lock_failed` | No | See `KeyListFragment.java:129-139` |
| **Drop mid-transfer** | Same `catchOnUi` | Same generic Toast | No | **This is the dangerous case** — see below |
| User cancel (15 s timeout) | `CancellationToken` | Background stays transparent | No | See `KeyItemAdapter.java:117` |
| Non-Ok `CommandResult` | switch fall-through | Toast, TODO for specific messages | No | See `KeyListFragment.java:116-123` |
| Credential expired | `SampleTokenRefreshHandler.refreshAuthenticationAsync` | Silent (SDK auto-refresh) | Yes, automatic | See [UC8](../uc8-token-refresh.md), [SC4](sc4-token-exchange.md) |
| Refresh fails | `onRefreshFailed` → `LoginActivity` | Kicked back to login | Manual (user re-authenticates) | See `SampleTokenRefreshHandler.java:46-59` |

---

## What "drop mid-transfer" actually means

`ConnectionLostException` (defined in `net.tpky.mc.ble`, thrown by the SDK, caught by the app):

```
User taps Trigger
  ↓
BLE GATT connect
  ↓
TLCP session opens
  ↓
TriggerLockCommand sent   ←── phone walks away, GATT drops
  ↓
????????                  ←── did the lock receive it? did it execute?
```

**The sample app cannot distinguish "command never sent" from "command sent, ack lost".** Both surface as the same generic `Toast(trigger_lock_failed)`. There is no retry, no idempotency probe, no state reconciliation.

The newer template distinguishes `ConnectionLostException` from `BleException` at the message level (see [ref-message-resolver.md](../ref-message-resolver.md#exception-types--transport-layer-failures)), but still does **not** retry or reconcile.

**For the parent project this is the exact TK3 research question.** Under WBSO TK3 we owe an answer to: "given the relay is unreliable and untrusted, how does the device stay atomic and idempotent?" Neither Tapkey codebase answers it — that gap is our contribution.

---

## Timeout behaviour

Only one timeout exists at the app layer:

- **15 seconds** on the entire `triggerLock` promise (`KeyItemAdapter.java:117`):
  ```java
  CancellationTokens.withTimeout(cts.getToken(), 15000)
  ```

If the SDK hasn't completed connect + TLCP session + command + response within 15 s, the token cancels and the operation fails. **The SDK's internal per-step timeouts are opaque.** No app-visible knobs for:
- Scan timeout (scan runs until fragment paused)
- GATT connect timeout
- MTU / service discovery timeout
- TLCP session establishment timeout
- Individual characteristic write timeout

For the PWA relay, all of these will be app-visible and must be explicitly bounded — Web Bluetooth does not provide defaults for most of them.

---

## Permission-denial paths

Documented fully in [UC4](../uc4-ble-permissions.md). Two branches:

1. **First denial** — Snackbar with rationale, re-prompt
2. **Permanent denial** — Snackbar → `ACTION_APPLICATION_DETAILS_SETTINGS` intent

**No graceful degradation.** Without BLE permissions, the whole key list becomes triggerless. The user can still view keys (server data is unaffected) but cannot unlock anything.

The parent project's PWA has an analogous constraint: without user gesture + Web Bluetooth permission, `requestDevice()` cannot even show the picker. Design assumption should be: **either the user grants BLE, or the relay is inert.**

---

## Error-taxonomy reference

For the full space of `CommandResultCode` and `ValidityError` values (as documented in the newer template's `MessageResolver.kt`), see [ref-message-resolver.md](../ref-message-resolver.md). Key protocol-layer signals worth naming in the parent project:

- **`Unauthorized`** — grant valid, but not for this lock/time (policy-layer reject; relay delivered successfully)
- **`LockDateTimeInvalid`** — clock skew; the device rejected because *its* clock disagreed with the grant window
- **`SessionBroken`** — TLCP session died; equivalent to `ConnectionLostException` at a lower layer
- **`ConcurrencyError`** — the lock saw two commands in flight (relevant to idempotency)
- **`DifferentDevice`** — the lock rejected because the phone identity didn't match

The parent project's protocol should have equivalents for at least: unauthorized, expired, session-broken, and idempotency-collision.
