# SC9 — One-Page Notes: GATT Structure & Scan/Connect/Error Flow

Implementation-agnostic synthesis for the PWA relay design. Everything below is language-, SDK-, and platform-neutral — only patterns, not code.

> **Concepts:** See [BLE-CONCEPTS.md](../BLE-CONCEPTS.md) for Central/Peripheral roles and Service UUID fundamentals referenced throughout.

---

## Scan

- **One BLE service UUID identifies the device family.** All devices from the same vendor/system advertise the same service UUID; environment separation (prod vs. sandbox) is done with a *different* UUID.
- **Scan filter is UUID-based, not name-based, not MAC-based.** MAC addresses are ephemeral (Android randomises them); names are user-editable. UUID is the only stable identifier at the scan layer.
- **RSSI/proximity is a derived signal**, not a filter — the app decides what "nearby" means, downstream of the raw scan.
- **Advertisement payload** may carry an identity element (Tapkey uses their "V1 advertising format" — the payload contents are proprietary). The parent project must define this payload explicitly.

## Picker

- **The picker is app-owned, not OS-owned** in the Android SDK model. This won't port to Web Bluetooth — the browser will always show the OS picker on `requestDevice()`. Design around this: the PWA either lets the user pick per-connection, or uses `getDevices()` for previously-authorised devices.
- **The picker filters the *authorised* key list by *proximity*.** Two independent data sources — server (which keys) and BLE (which are nearby) — merge in the UI.

## Connect

- **Connect happens per-command.** Every unlock is a fresh scan → connect → command → disconnect cycle, not a persistent connection.
- **The BLE MAC is looked up from the stable device ID** immediately before connecting; the connect call takes the MAC, not the stable ID.
- **Service discovery, MTU negotiation, characteristic subscription** all happen inside the SDK. The PWA must do these explicitly: `getPrimaryService`, `getCharacteristic`, `startNotifications`.

## Present (credential exchange)

- **The credential is server-signed, cached locally, and presented to the device over the BLE session.** The device verifies the signature offline using its embedded trust anchor.
- **Time restrictions are enforced on the device**, not on the phone. The phone's clock is untrusted.
- **The wire protocol above GATT is a custom framing layer** (Tapkey's TLCP). The parent project defines its own equivalent — must specify write characteristic, notify characteristic, framing format, encryption (if any), and session establishment sequence.

## Result

- **The device returns a typed result code**, not a generic ok/fail. At minimum, categories should include: authorised & executed, unauthorised, expired, clock-mismatch, concurrency-collision, protocol-error.
- **Result codes carry severity** — some are user-actionable warnings ("wrong time"), some are hard errors ("hardware failure"). The parent project should classify accordingly.
- **Non-Ok results still mean "the command reached the device"** — this is different from a transport-layer failure, which may leave delivery ambiguous.

## Disconnect

- **Implicit disconnect on scope exit** in the Android SDK model (`executeCommandAsync` returns → SDK closes GATT). The template shows one explicit pattern: `runAndDisconnectAsync { ... }` for NFC.
- **Web Bluetooth needs explicit `disconnect()`**. The parent project should always disconnect explicitly to release the GATT slot (Web Bluetooth has a per-origin limit).

## Error taxonomy

Three orthogonal layers:

1. **Permission layer** — user hasn't granted BLE/location; no scanning possible. Recovery: OS-mediated re-prompt or settings link.
2. **Transport layer** — connection failed, dropped mid-command, timed out. This is the *ambiguous delivery* case. Recovery must be designed at the protocol layer, not the transport layer.
3. **Protocol layer** — command reached the device and got a typed response (Ok or specific rejection). Recovery is specific to the code.

**The transport-layer ambiguity is the TK3 research question.** Neither Tapkey codebase resolves it — both treat drop-mid-command as an unrecoverable app-level error.

## Timeouts

- **One app-level end-to-end timeout** (15 s in Tapkey's UI). Everything below it is SDK-internal and invisible.
- **The PWA must define timeouts explicitly at every layer:** scan timeout, connect timeout, service discovery, per-characteristic write, per-notify wait, overall command budget. Web Bluetooth has no defaults for most of these.

## Concurrency & session state

- **One command at a time per lock.** The SDK does not queue concurrent commands to the same device. Two rapid taps produce a `ConcurrencyError` (documented in the template's `ValidityError` enum).
- **No persistent state across commands.** Each unlock is independent — no session cookies, no in-progress state that survives a disconnect.
- **The lock is authoritative about session state.** If the lock sees a second command mid-first-command, it rejects the second, not the first.

## What must be pre-agreed between server, phone, and device

1. Stable device identifier format (Tapkey's `physicalLockId`)
2. Grant/token format including signature, validity window, target device ID
3. Service UUID (per environment)
4. Characteristic UUIDs (write + notify at minimum)
5. Advertisement payload format
6. Wire protocol above GATT: framing, encryption, session semantics
7. Result code enum with severity classification
8. Error taxonomy across permission / transport / protocol layers

The Tapkey sample app gives us evidence that (1)–(2) are decoupled from (3)–(6), and (3)–(6) are decoupled from (7)–(8). That decoupling is itself a design insight — each concern can be specified and tested independently.
