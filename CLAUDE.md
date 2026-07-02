# CLAUDE.md — android-sample-app Study & Reference Notes

This repo is a **study reference only** — cloned 2026-07-02 from the original Tapkey Android SDK sample app (Apache-2.0, Java, v2.19.2.2, officially discontinued).

**Do not build or ship from this repo.** It exists to extract BLE architecture patterns (GATT structure, scan flow, error handling) that inform the PWA relay design in the parent project (`../Secure-datasharing-platform`).

---

## Purpose

The parent project specifies a **PWA relay (Web Bluetooth API)** as the funded research path — this keeps BLE protocol handling transparent and observable at the app layer. The android-sample-app serves as a **reference for how BLE conversations are structured**, not as a code baseline to port.

**What to study:**
- GATT service discovery and characteristic reads/writes
- BLE device scan filters and picker UX
- Connection lifecycle: scan → connect → present credential → receive result → disconnect
- Error handling: scan timeout, connection drop, user cancel, permission denial

**What NOT to do:**
- Copy Tapkey SDK calls (e.g., `TapkeyTokenExchangeManager`, `ApplicationGrantDto`)
- Port the OAuth / token-refresh logic (our server-side token contract is different)
- Reuse the Android-specific BLE APIs (we build on Web Bluetooth instead)

---

## Study Checklist

The parent project's Phase 1 plan includes these study tasks:

- [ ] Read `App.java` and `MainActivity.java` — app lifecycle entry point
- [ ] Read `KeyListFragment.java` + `KeyItemAdapter.java` — device list UI pattern
- [ ] Read `SampleServerManager.java` — credential fetch before BLE (pattern only, not details)
- [ ] Read `AuthStateManager.java` + `SampleTokenRefreshHandler.java` + `TapkeyTokenExchangeManager.java` — token exchange sequence (note the dance, ignore SDK specifics)
- [ ] Read `ApplicationGrantDto.java` — credential/grant data model shape
- [ ] Map full interaction: scan → picker → connect → present → result → disconnect; note which steps are SDK-opaque
- [ ] Identify GATT service UUID(s); note scan-filter pattern
- [ ] Document error paths: scan timeout, connect fail, credential reject, drop mid-transfer
- [ ] Write one-page notes (scratch only, not committed) on GATT structure and scan/connect/error flow in implementation-agnostic terms
- [ ] Cross-check notes against parent project's GATT contract: `requestDevice()` with service UUID filter, no pairing/bonding, protocol client-agnostic

---

## Project Context

**Parent project:** `../Secure-datasharing-platform`

- **What it is:** Secure offline IoT bridge — server signs one-time tokens → relayed by untrusted phone over BLE → verified offline by ESP32-S3 device
- **Research question (TK3):** Device-side atomicity and idempotency under an unreliable, untrusted relay
- **Why PWA, not native Android SDK:** WBSO constraint. Native SDK would move BLE into a platform-managed stack; PWA keeps relay thin and protocol-transparent so we can prove the protocol works, not just that the SDK handles it
- **iOS relay:** Deferred to Phase 4 (Safari has no Web Bluetooth support)

**Read:** `../Secure-datasharing-platform/ADR/adr-0005.md` for the full rationale.

---

## Use Case Documentation

Full use case diagrams (class + sequence + explanations) for each feature:
→ **[docs/README.md](docs/README.md)**

---

## Reference: Key Java Files

| File | Role | Study goal |
| --- | --- | --- |
| `App.java` | Application lifecycle | Entry point, when does BLE init happen |
| `MainActivity.java` | Main UI activity | Where user interacts; what triggers scan/connect |
| `KeyListFragment.java` | Device list view | How BLE devices are surfaced to user; picker UX |
| `KeyItemAdapter.java` | List item binding | Device name / UUID rendering |
| `SampleServerManager.java` | Server credential fetch | Pattern: when does app ask server for permission before BLE |
| `AuthStateManager.java` | Auth state mgmt | Token lifecycle; when tokens are requested / refreshed |
| `SampleTokenRefreshHandler.java` | Token refresh | Handles 401 / token expiry |
| `TapkeyTokenExchangeManager.java` | SDK token exchange | **Ignore SDK calls; note the flow shape only** |
| `ApplicationGrantDto.java` | Credential model | Data structure of what the server returns |

---

## Known Limitations

- **Outdated.** Tapkey has moved to "Tapkey App Template for Android" as the official reference. This sample is officially discontinued but still valuable as a concrete BLE + token-flow example.
- **Gradle / SDK dependencies** may need updates to build against the latest Android SDK (API 35+). Not required for study purposes.
- **Tapkey SDK specifics** (OAuth flows, vendor-proprietary extensions) do not apply to our system; extract only the BLE architecture.

---

## Notes Workspace

(Scratch notes from study — not committed. Delete or move to parent project docs as findings settle.)

```
[placeholder for your notes as you read the code]
```
