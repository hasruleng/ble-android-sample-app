# Study Checklist — Findings

Each file completes one item from the [CLAUDE.md](../../CLAUDE.md) study checklist. Findings are for the parent project's ([Secure-datasharing-platform](../../../Secure-datasharing-platform)) PWA relay design.

| # | Study task | File |
|---|-----------|------|
| SC1 | App lifecycle entry point (`App.java` + `MainActivity.java`) | [sc1-app-lifecycle.md](sc1-app-lifecycle.md) |
| SC2 | Device list UI pattern (`KeyListFragment` + `KeyItemAdapter`) | [sc2-device-list-ui.md](sc2-device-list-ui.md) |
| SC3 | Server credential fetch before BLE (`SampleServerManager`) | [sc3-credential-fetch.md](sc3-credential-fetch.md) |
| SC4 | Token exchange dance (`AuthStateManager` + refresh + exchange) | [sc4-token-exchange.md](sc4-token-exchange.md) |
| SC5 | Grant data model (`ApplicationGrantDto`) | [sc5-grant-data-model.md](sc5-grant-data-model.md) |
| SC6 | Full interaction map (scan → picker → connect → present → result → disconnect) with SDK-opacity marked | [sc6-interaction-map.md](sc6-interaction-map.md) |
| SC7 | GATT service UUID(s) + scan-filter pattern | [sc7-gatt-uuid-scan-filter.md](sc7-gatt-uuid-scan-filter.md) |
| SC8 | Error paths (scan timeout, connect fail, credential reject, drop mid-transfer) | [sc8-error-paths.md](sc8-error-paths.md) |
| SC9 | Implementation-agnostic one-page notes | [sc9-one-page-notes.md](sc9-one-page-notes.md) |
| SC10 | Cross-check against ADR-0005 GATT contract | [sc10-cross-check-adr.md](sc10-cross-check-adr.md) |

---

## Prerequisites

**New to BLE?** Start with [BLE-CONCEPTS.md](../BLE-CONCEPTS.md) — it explains Central vs. Peripheral roles, Service UUID fundamentals, and GATT hierarchy that underpin all findings below.

---

## Read in this order for a first pass

**Fast overview:** SC9 (one-page notes) → SC10 (ADR cross-check).

**Detailed dive:** SC1 → SC2 → SC3 → SC4 → SC5 (mechanics), then SC6 → SC7 → SC8 (BLE-specific), finish with SC9 → SC10 (synthesis).

## Key discoveries

- **SC7** — The Tapkey BLE service UUID has ASCII `net.tpky` as its first 8 bytes. Prod vs. sandbox use distinct UUIDs.
- **SC6** — The security-critical parts (present + result decoding) are fully SDK-opaque in the sample app. The newer template exposes the service UUID but not the wire protocol.
- **SC8** — Neither Tapkey codebase implements retry or reconnect on transport-layer failure. This gap is the parent project's TK3 contribution.
- **SC10** — Every item in ADR-0005's GATT contract is validated by Tapkey practice.
