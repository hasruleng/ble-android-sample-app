# SC7 — GATT Service UUID & Scan Filter Pattern

**Question:** What is the service UUID the Tapkey lock advertises? What scan-filter pattern does the app use?

**This is the pivotal finding for the parent project's Web Bluetooth relay design.**

> **Background:** New to BLE concepts? Start with [BLE-CONCEPTS.md](../BLE-CONCEPTS.md) to understand Central vs. Peripheral roles and why Service UUID is the stable identifier for scanning.

---

## In the sample app: nothing visible

Grep across [this repo](../../app/src/main/) for `UUID`, `ParcelUuid`, `ScanFilter`, `characteristic`, `GATT`, `advertis`:

- **AndroidManifest.xml** — declares BLE permissions only (`BLUETOOTH_SCAN`, `BLUETOOTH_CONNECT`, `ACCESS_FINE_LOCATION`), no UUIDs
- **All Java files** — zero matches for service UUIDs, characteristic UUIDs, or scan filters

The sample app is entirely dependent on `BleLockScanner.startForegroundScan()` to configure the scan on its behalf. The service UUID is baked into the Tapkey `.aar` binary distribution.

---

## In the newer template: the UUID is exposed

Grep across [`tapkey-keyring-app-template-android`](../../../tapkey-keyring-app-template-android/):

`app/app/src/prod/res/values/config.xml:6`:
```xml
<string name="tapkey_v1_ble_service_uuid" translatable="false">6e65742e-7470-6b79-2ea0-000006060101</string>
```

`app/app/src/sandbox/res/values/config.xml:6`:
```xml
<string name="tapkey_v1_ble_service_uuid" translatable="false">6e65742e-7470-6b79-2ea0-000006060142</string>
```

Wired up in `App.kt:77-93`:
```kotlin
val tapkeyBleAdvertisingFormatBuilder = TapkeyBleAdvertisingFormatBuilder()
val v1BleServiceUuid = resources.getString(R.string.tapkey_v1_ble_service_uuid)
if (!TextUtils.isEmpty(v1BleServiceUuid)) {
    tapkeyBleAdvertisingFormatBuilder.addV1Format(v1BleServiceUuid)
}
val tapkeyBleAdvertisingFormat = tapkeyBleAdvertisingFormatBuilder.build()
// passed to TapkeyServiceFactoryBuilder.setBluetoothAdvertisingFormat(...)
```

---

## Decoding the UUID

UUIDs are 128-bit values usually opaque. Tapkey's prefix is not opaque:

```
6e65742e-7470-6b79-2ea0-000006060101
└──┬──┘ └──┬──┘ └─┬─┘
   │       │      │
   │       │      └─ 6b79      = "ky"  (ASCII)
   │       └─ 7470             = "tp"  (ASCII)
   └─ 6e65742e                 = "net." (ASCII)

First 8 bytes read as ASCII: "net.tpky"
```

**The service UUID prefix literally spells `net.tpky` — Tapkey's Java package namespace.** The remaining bits (`2ea0-000006060101` for prod, `...0142` for sandbox) encode environment and (presumably) a protocol version marker.

The `V1` in `tapkey_v1_ble_service_uuid` and `addV1Format` suggests forward-compat design: future lock firmware may advertise a `v2` UUID, and the SDK can filter on both. The builder pattern (`.addV1Format(...)`) hints that a scanner can be configured for multiple UUIDs simultaneously.

---

## Scan filter pattern (inferred)

Neither codebase shows raw `ScanFilter` construction — it's inside the SDK — but from the API shape:

1. **UUID filter** — the scanner is configured with the service UUID(s) via `TapkeyBleAdvertisingFormatBuilder`
2. **No RSSI filter at the app layer** — proximity classification happens downstream (`isLockNearby()` returns a boolean the SDK computes)
3. **No name/manufacturer filter** — all Tapkey locks advertise the same service UUID(s); differentiation is by the payload in the advertisement

For Android BLE this maps to:
```java
ScanFilter filter = new ScanFilter.Builder()
    .setServiceUuid(ParcelUuid.fromString("6e65742e-7470-6b79-2ea0-000006060101"))
    .build();
BluetoothLeScanner.startScan(List.of(filter), settings, callback);
```

**The MAC address is NOT a filter target.** The lock's MAC is resolved *after* scanning, via `BleLockScanner.getLock(physicalLockId).getBluetoothAddress()` — see [SC5](sc5-grant-data-model.md).

---

## Direct implications for the PWA Web Bluetooth relay

Web Bluetooth's `navigator.bluetooth.requestDevice()` takes filters that mirror this pattern:

```javascript
const device = await navigator.bluetooth.requestDevice({
    filters: [
        { services: ['6e65742e-7470-6b79-2ea0-000006060101'] }  // prod
        // { services: ['6e65742e-7470-6b79-2ea0-000006060142'] }  // sandbox
    ],
    optionalServices: [/* any additional service UUIDs for read/write */]
});
```

**⚠️ Caveats before copying the UUID literally:**

1. **The parent project has its own device firmware.** These UUIDs are Tapkey's — the ESP32-S3 will advertise a *different* UUID chosen by the parent project. This finding is a **pattern reference**, not a copy target.

2. **Web Bluetooth requires the service UUID to be in `filters[].services` OR `optionalServices` before `getPrimaryService()` will work.** Missed configuration = silent failure. Match the Android manifest-permission pattern: enumerate every UUID the app might touch.

3. **The "namespace-encoded-into-UUID" pattern is worth borrowing.** For the parent project, consider encoding a project namespace prefix (e.g., ASCII "sdsp." → hex) in the UUID so devices are self-identifying under inspection. This makes protocol reverse-engineering easier during debugging without giving attackers useful information (the UUID is public by design).

4. **Version marker in UUID.** Tapkey's `V1` naming plus a distinct sandbox UUID is a real design pattern: separate UUIDs for staging vs. prod prevents dev phones from accidentally triggering prod locks. Parent project should adopt this — one UUID per environment.

---

## What we still don't know

The scan filter is now clear. **Characteristic UUIDs are still opaque** — neither codebase declares them. The SDK internally knows which characteristics carry TLCP write, TLCP notify, and OTA/DFU streams. For the PWA relay, these will be defined by the parent project's own firmware and must be documented up front.

Recommended action for the parent project: define a `GATT-CONTRACT.md` naming:
- Service UUID (per environment)
- Characteristic UUIDs (TLCP-equivalent write + notify)
- Advertisement payload format (manufacturer data, if any)
- Version marker in advertisement / service UUID

before any firmware or relay code is written.
