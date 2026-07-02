# SC5 — Grant Data Model

**Question:** What data structure does the server return for a grant? What fields inform the BLE flow?

**File:** [ApplicationGrantDto.java](../../app/src/main/java/net/tpky/demoapp/ApplicationGrantDto.java)

---

## Fields

| Field | Type | Optional? | Semantics |
|-------|------|-----------|-----------|
| `id` | String | required | Unique grant ID (also used as `grantIds[]` query param in [SC3](sc3-credential-fetch.md)) |
| `state` | String | optional | Grant state (e.g. `"active"`) |
| `validBefore` | Date | optional | Expiry timestamp (ISO 8601) |
| `validFrom` | Date | optional | Validity start timestamp (ISO 8601) |
| `timeRestrictionIcal` | String | optional | iCalendar-format recurrence (e.g. "Mon–Fri 9–17") |
| `issuer` | String | optional | Issuing organization display name |
| `granteeFirstName` | String | optional | Recipient's first name |
| `granteeLastName` | String | optional | Recipient's last name |
| `lockTitle` | String | optional | Human-readable lock name |
| `lockLocation` | String | optional | Lock location string |
| **`physicalLockId`** | String | required | **Hardware lock ID — used for BLE matching** |

Parsed from JSON in [SampleServerManager.java:129-145](../../app/src/main/java/net/tpky/demoapp/SampleServerManager.java) using `SimpleDateFormat("yyyy-MM-dd'T'HH:mm:ss.SSSXXX")` for date fields.

---

## The important one: `physicalLockId`

Everything else is UI metadata. `physicalLockId` is the join key between:

- **Grant list** (server-issued, cloud-authoritative)
- **BLE scan results** (`BleLockScanner.isLockNearby(physicalLockId)`, `BleLockScanner.getLock(physicalLockId).getBluetoothAddress()`)
- **TLCP command context** (`bleLockCommunicator.executeCommandAsync(bleAddr, physicalLockId, ...)`)

The BLE MAC address is **never used** as an identity anchor at the app layer — only for the low-level transport call. The lock is identified by its `physicalLockId`, which is stable across MAC randomisation, hardware swaps, etc.

**This is the key architectural pattern for the parent project:** the physical device carries a stable ID that the cloud, the relay, and the device firmware all agree on, and the transport address (BLE MAC) is treated as ephemeral.

---

## Time-restriction fields

`validFrom`, `validBefore`, and `timeRestrictionIcal` are **client-side hints only**. The Tapkey lock verifies validity itself using its internal clock (`LockDateTimeInvalid` is a real error code — see [ref-message-resolver.md](../ref-message-resolver.md)). The app displays these fields for UX; enforcement is offline on the device.

Same pattern applies to the parent project: **the ESP32-S3 must not trust the phone's clock.** Time restrictions must be encoded in the signed token and verified against the device's own clock or a signed timestamp.

---

## PWA relay implication

Design the parent project's grant/credential DTO with:
- One stable device ID (equivalent to `physicalLockId`)
- Server-issued validity window (equivalent to `validFrom` / `validBefore`)
- Cryptographic signature covering both (Tapkey's SDK does this internally; ours needs to be explicit)
- UI-only metadata (title, location, issuer) — clearly separated from security-critical fields

Do **not** carry:
- Basic Auth-related fields (username, password fragments)
- Anything client-editable that the device would trust
