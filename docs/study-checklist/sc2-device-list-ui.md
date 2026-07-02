# SC2 — Device List UI / Picker Pattern

**Question:** How are BLE devices surfaced to the user? What triggers the trigger button? How does the picker work?

**Files:** [KeyListFragment.java](../../app/src/main/java/net/tpky/demoapp/KeyListFragment.java), [KeyItemAdapter.java](../../app/src/main/java/net/tpky/demoapp/KeyItemAdapter.java)

---

## There is no OS-level BLE picker

**Key finding:** The user never sees a system BLE device picker. Instead:

1. The app queries the Tapkey backend for the list of **keys the user has** (a logical list, not a scan result)
2. The BLE scanner runs continuously in the background
3. Each row in the list checks `isLockNearby(physicalLockId)` — a boolean flag maintained by the scanner
4. Only rows with `isLockNearby == true` render a Trigger button

The "picker" is therefore **the whole key list**, filtered by proximity in real time. This is very different from Web Bluetooth's model, where `requestDevice()` shows an OS-mediated picker on every call.

---

## Data flow into the adapter

`KeyListFragment.onKeyUpdate()` at `KeyListFragment.java:273-321`:

```
keyManager.queryLocalKeysAsync(userId)                    // 1. local key store
  → List<KeyDetails>
     ↓
sampleServerManager.getGrants(u, p, grantIds[])           // 2. enrich w/ metadata
  → List<ApplicationGrantDto>                                (see SC5)
     ↓
zip by grantId
  → List<Tuple<KeyDetails, ApplicationGrantDto>>          // 3. joined type
     ↓
adapter.clear() + adapter.addAll(listItems)               // 4. push to UI
```

`KeyItemAdapter` extends `ArrayAdapter<Tuple<KeyDetails, ApplicationGrantDto>>` (`KeyItemAdapter.java:24`).

---

## The Handler indirection

`KeyItemAdapter.java:28-32` defines:

```java
public interface KeyItemAdapterHandler {
    boolean isLockNearby(String physicalLockId);
    Promise<Boolean> triggerLock(String physicalLockId, CancellationToken ct);
}
```

Implemented at `KeyListFragment.java:87-141`:
- `isLockNearby` → delegates to `bleLockScanner.isLockNearby(physicalLockId)`
- `triggerLock` → resolves BLE address, calls `bleLockCommunicator.executeCommandAsync` (see [UC5](../uc5-trigger-lock.md))

**Pattern insight:** the adapter is the **only** class that talks about BLE, but it doesn't know anything about BLE. It just asks "is X nearby?" and "please trigger X". This decouples list rendering from BLE state — good template for the PWA relay.

---

## Trigger button visibility

`KeyItemAdapter.java:96-154`:

```java
if (!handler.isLockNearby(grant.getPhysicalLockId())) {
    triggerButton.setVisibility(GONE);
    cancelButton.setVisibility(GONE);
    // → row shows only "not in range" state
} else {
    triggerButton.setVisibility(VISIBLE);
    // → user can tap
}
```

---

## What triggers re-render

`KeyListFragment.java:221`:

```java
bleObserverRegistration = bleLockScanner
    .getLocksChangedObservable()
    .addObserver(stringBleLockMap -> adapter.notifyDataSetChanged());
```

Every time the scanner detects a new lock (or loses one), the whole list re-evaluates `isLockNearby()` for every row. This drives the button-appearing / button-disappearing UX as the user walks toward / away from a lock.

A second observer (`keyManager.getKeyUpdateObservable`) refreshes the *underlying key data* when the Tapkey backend reports a new grant. These two observers are independent and merge in the adapter.

---

## PWA relay implication

For the parent project (Web Bluetooth):
- Web Bluetooth does not have a persistent-scan observable equivalent. `navigator.bluetooth.getDevices()` returns previously-authorised devices; `requestDevice()` shows a one-shot OS picker.
- The Tapkey model ("scan continuously in background, filter list by proximity") **does not port**. Web Bluetooth intentionally forces a per-connection user gesture.
- Design decision: the PWA either (a) shows the full key list and requires the user to explicitly tap "connect" per key (triggering `requestDevice()`), or (b) uses [`experimentalScan`](https://developer.mozilla.org/en-US/docs/Web/API/Bluetooth/requestLEScan) where available (Chromium desktop only, flagged). Option (a) is the safer default.
