# SC10 — Cross-Check Against Parent Project's GATT Contract

**Question:** Do the sample-app findings validate or contradict the parent project's ADR-0005 GATT contract requirements?

**Reference:** [ADR-0005](../../../Secure-datasharing-platform/ADR/adr-0005.md) — iOS/Web Bluetooth constraint and PWA-first BLE strategy.

> **Concepts:** Unfamiliar with GATT terminology? See [BLE-CONCEPTS.md](../BLE-CONCEPTS.md) for Central/Peripheral roles and Service UUID fundamentals.

---

## ADR-0005 stipulations (the "GATT contract" line-item)

From ADR-0005, the parent project's GATT contract requirements are:

1. **`requestDevice()` with service UUID filter** — the Web Bluetooth entry point must be UUID-filtered, not name-based or MAC-based
2. **No pairing / no bonding** — the phone must not build a persistent BLE bond with the device
3. **Protocol client-agnostic** — no Android- or iOS-specific assumptions; the same wire protocol must work from Android PWA (Phase 1–3), iOS App Clip or native (Phase 4+), and any future relay

---

## Cross-check: what the sample app tells us

### ✅ Item 1 — UUID filter

**Validated.** [SC7](sc7-gatt-uuid-scan-filter.md) shows that Tapkey uses a **service-UUID-based scan filter** as the sole discovery mechanism. The newer template exposes the UUID (`6e65742e-7470-6b79-2ea0-000006060101` prod, `...0142` sandbox) and passes it into the SDK's advertising-format builder. No name filter, no MAC filter.

This confirms the ADR's `requestDevice({ filters: [{ services: [UUID] }] })` approach is the canonical pattern — not an ADR quirk.

**Design carry-over for parent project:** define the ESP32-S3 service UUID early, register it in the PWA's `filters` and `optionalServices`, and separate prod from sandbox with distinct UUIDs (Tapkey's split pattern).

---

### ✅ Item 2 — No pairing / no bonding

**Validated.** Grep for `createBond`, `bondState`, or `BluetoothDevice.BOND_` across both Tapkey codebases returns zero matches. The Tapkey model is:

- Scan by service UUID
- Connect GATT (unbonded)
- Present cryptographic credential over TLCP (app-layer trust, not link-layer)
- Execute command
- Disconnect

**No BLE bond is ever created.** All authentication is at the app layer — the cryptographic material in the grant + TLCP session establishment.

This matches the parent project's model exactly: server signs a one-time token → phone relays it → device verifies signature offline. The BLE link itself is treated as an untrusted channel.

**Design carry-over:** the parent project's protocol must not depend on any link-layer security (Just Works pairing, LE Secure Connections, or bonded keys). All security is above GATT.

---

### ⚠️ Item 3 — Protocol client-agnostic

**Partially validated, with a caveat.**

The Tapkey model is **client-agnostic in principle** (the lock accepts commands from any client presenting a valid signed credential over TLCP), but **client-specific in practice** because:

- TLCP is Tapkey's proprietary wire protocol; only Tapkey's SDKs speak it
- Vendor SDKs exist for Android/iOS/Cordova/etc. — Tapkey ships bindings so third parties don't have to reimplement TLCP
- **No Web Bluetooth SDK** — this is a real gap in Tapkey's coverage, forcing PWAs to either wrap the Android SDK (impossible) or reimplement TLCP (undocumented)

For the parent project this is a **positive signal**: our protocol is designed by us, so we simply do not create the same gap. The wire protocol above GATT should be:
- Fully documented in a `GATT-CONTRACT.md`
- Reference-implemented in a small library (not a full SDK)
- Testable with generic BLE tools (`nRF Connect`, `bleak`, `noble`, etc.)

**Design carry-over:** avoid proprietary wire protocols. If the parent project's protocol above GATT can be described in a spec that a competent BLE dev can implement from scratch, then Android PWA, iOS App Clip, iOS native, Python test harness, and future clients all have equal footing.

---

## Additional cross-checks (not in ADR-0005 but relevant)

| Sample-app finding | Parent-project implication |
|--------------------|----------------------------|
| Stable `physicalLockId` decouples from ephemeral BLE MAC ([SC5](sc5-grant-data-model.md)) | ✅ Adopt the same pattern: stable device ID in the signed token, MAC used only for the GATT connect |
| Server credential fetch happens **before** any BLE call ([SC3](sc3-credential-fetch.md)) | ✅ Same ordering: PWA fetches signed token from server first, only then connects to device |
| Silent token refresh via a registered handler ([SC4](sc4-token-exchange.md)) | ⚠️ Not applicable — parent project uses one-shot tokens, not refreshable access tokens |
| Time restrictions verified **on the device**, not phone ([SC5](sc5-grant-data-model.md)) | ✅ Same constraint: ESP32-S3 must not trust phone clock |
| Transport-layer errors (`ConnectionLostException`) give ambiguous delivery ([SC8](sc8-error-paths.md)) | ⚠️ **Unresolved in Tapkey — this is the TK3 research question** for the parent project |
| No retry / reconnect logic in either Tapkey codebase ([comparison-error-handling.md](../comparison-error-handling.md)) | ⚠️ Must be designed from scratch — no reference to borrow |
| Result code enum with severity classification ([ref-message-resolver.md](../ref-message-resolver.md)) | ✅ Adopt the pattern: typed result codes + severity, not free-text errors |

---

## Recommendation to update ADR-0005 or downstream ADRs

Based on this study, the following items are candidates for a follow-up ADR:

1. **Service UUID assignment** — one UUID per environment; version marker in UUID or advertisement payload
2. **Characteristic layout** — write + notify at minimum; document each characteristic's role
3. **Advertisement payload** — what identity information (if any) the device advertises before connection
4. **Idempotency and reconciliation protocol** — the TK3 research contribution; must specify what the device does when it sees the same command twice, and how the relay learns the outcome after a drop

Items 1–3 unblock firmware development. Item 4 is the research artifact.

---

## Conclusion

**The GATT contract in ADR-0005 is consistent with observed Tapkey practice.** Every checklist item — UUID filter, no bonding, client-agnostic — either has direct evidence in the Tapkey codebases or is orthogonal (client-agnostic is *aspirational* for Tapkey, *achievable* for us).

The Tapkey study delivers everything it was chartered to deliver:
- ✅ Confirmed the scan/connect/present/result/disconnect shape
- ✅ Identified the service-UUID pattern including environment separation
- ✅ Documented the error taxonomy (via the newer template's `MessageResolver.kt`)
- ✅ Confirmed no link-layer bonding
- ⚠️ Confirmed no reference for retry/reconnect — that is our contribution to make

Phase 1 study is complete. Phase 2 can begin protocol drafting with the above patterns as validated priors.
