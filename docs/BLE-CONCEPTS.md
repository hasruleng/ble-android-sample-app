# BLE Concepts Reference

**Purpose:** Clarify fundamental BLE architecture so the study checklist findings (SC1–SC10) make sense.

---

## Central vs. Peripheral (and which is which)

In any BLE conversation, there are **two sides**:

| Role | Device | In your case |
|------|--------|-------------|
| **BLE Central** | Initiates scan, connects, reads/writes | **Your phone or PWA** |
| **BLE Peripheral** | Advertises, accepts connections, hosts data | **The BLE lock or ESP32-S3 device** |

```
┌─────────────────────────────────────────────────────────────┐
│ Your Phone / PWA (BLE Central)                              │
│                                                              │
│  1. "Scan for BLE devices"                                  │
│  2. Find devices advertising Service UUID "net.tpky"        │
│  3. Connect to a lock                                        │
│  4. Read/write data to the lock's characteristics           │
└─────────────────────────────────────────────────────────────┘
                              ↕ (Bluetooth LE connection)
┌─────────────────────────────────────────────────────────────┐
│ BLE Lock or ESP32-S3 (BLE Peripheral)                       │
│                                                              │
│  1. Advertises: "I exist! I offer service UUID net.tpky"   │
│  2. Accepts incoming connections from phone                 │
│  3. Hosts data in Services → Characteristics                │
│  4. Responds to read/write requests from phone              │
└─────────────────────────────────────────────────────────────┘
```

---

## Service UUID — what it identifies

A **Service UUID** is a 128-bit label that the **peripheral (lock) advertises** in its BLE advertisement packet.

Think of it like a storefront sign:

```
Lock's BLE Advertisement Packet
┌────────────────────────────────────────┐
│ "Hey, I'm here!"                       │
│                                        │
│ Service UUIDs I offer:                 │
│ • 6e65742e-7470-6b79-...  (net.tpky)  │
│   ↑                                    │
│   This is a Tapkey lock                │
└────────────────────────────────────────┘
```

When your **phone (central) scans**, it receives these advertisement packets and filters them:

```
Phone scanning:
┌────────────────────────────────────────┐
│ "Find all devices advertising          │
│  Service UUID = 6e65742e-7470-6b79-... │
│  (net.tpky)"                           │
└────────────────────────────────────────┘
                              ↓
         Returns only Tapkey locks nearby
         (filters out all other BLE devices)
```

---

## Why NOT use MAC or name?

The peripheral (lock) has other identifiers that **change or are unreliable**:

| Identifier | Stability | Why it fails as a filter |
|------------|-----------|-------------------------|
| **MAC Address** | Ephemeral | Android randomizes for privacy; changes per scan session |
| **Device Name** | User-editable | Owner can rename the lock; two locks might have same name |
| **Service UUID** | Stable | Baked into firmware; same across power cycles, scans, etc. |

**Example:**

```
Same lock, different scans:

Scan 1:
  MAC: AA:BB:CC:DD:EE:FF
  Name: "Front Door"
  Services: [net.tpky]

Scan 2:
  MAC: 11:22:33:44:55:66  ← different! (randomized)
  Name: "Front Door"
  Services: [net.tpky]       ← same!

Scan 3 (user renames):
  MAC: 99:88:77:66:55:44  ← different! (randomized again)
  Name: "Kitchen Lock"     ← different!
  Services: [net.tpky]       ← same!
```

**Solution:** Filter by Service UUID. It never changes.

---

## Classic Bluetooth vs. BLE Scanning (the shift in mental model)

If you've paired with Bluetooth devices before (headphones, speakers), the UX felt like this:

```
Classic Bluetooth (what you might be used to):
┌──────────────────────────────────────────┐
│ "Show all Bluetooth devices"             │
│                                          │
│ Available devices:                       │
│  • My Headphones                         │
│  • Car Audio                             │
│  • Kitchen Speaker                       │
│  • (other random devices)                │
│                                          │
│ You: Tap "My Headphones" → connects     │
└──────────────────────────────────────────┘
```

**BLE with Service UUID filtering changes this:**

```
BLE Scanning (the new way):
┌──────────────────────────────────────────┐
│ Phone auto-filters:                      │
│ "Show only devices advertising           │
│  Service UUID = net.tpky"                │
│                                          │
│ Nearby devices with that UUID:           │
│  • Front Door Lock                       │
│  • Kitchen Lock                          │
│  (everything else is invisible)          │
│                                          │
│ You: Tap "Front Door Lock" → connects   │
└──────────────────────────────────────────┘
```

**Key difference:**
- Classic: You search by device *name* from a big list
- BLE: Phone *automatically filters* to show only devices with the matching UUID
- **You don't search for the name; the UUID does the filtering for you**

**Why this matters:**
- Someone could rename their device to "Front Door Lock" (fake)
- But they **cannot fake the Service UUID** (it's baked in firmware)
- So UUID-based filtering is more secure and reliable than name matching

---

## Full data hierarchy inside a peripheral

Once the phone connects to a lock, it discovers this structure:

```
BLE Lock (Peripheral)
│
├─ Service: "net.tpky" (UUID: 6e65742e-7470-6b79-...)
│  │
│  ├─ Characteristic: TLCP Write  (UUID: ???)
│  │   └─ Permission: Write
│  │   └─ Data: your command bytes
│  │
│  └─ Characteristic: TLCP Notify (UUID: ???)
│      └─ Permission: Read + Notify
│      └─ Data: lock's response bytes
│
├─ Service: "Battery" (UUID: 0000180f-...)  [standard Bluetooth service]
│  │
│  └─ Characteristic: Battery Level (UUID: 00002a19-...)
│      └─ Data: 0-100 (%)
│
└─ Service: "Device Information" (UUID: 0000180a-...)
   └─ Characteristic: Manufacturer Name (UUID: 00002a29-...)
       └─ Data: "Tapkey"
```

**Key point:** Service UUID identifies *which service group you're talking to*, then Characteristic UUID identifies *which data within that service*.

For Tapkey:
- **Service UUID** `net.tpky` = "lock control service"
- **Characteristic UUIDs inside** are opaque (not exposed in the sample app, only in the SDK binary)

For the parent project's PWA relay:
- You'll define your own Service UUID
- You'll also define your own Characteristic UUIDs (e.g., one for commands, one for responses)

---

## Scanning flow (the phone's perspective)

```
User opens app
        ↓
Phone requests BLE scan permission from OS
        ↓
[User grants permission]
        ↓
Phone tells OS: "Scan for devices advertising Service UUID 6e65742e-7470-6b79-..."
        ↓
OS starts scanning (sends BLE scan requests into the air)
        ↓
Lock receives request and replies: "I'm here! Services: [net.tpky]"
        ↓
OS delivers advertisement packet to app
        ↓
App checks: "Does this packet advertise 6e65742e-7470-6b79-...? Yes → add to list"
        ↓
User sees lock in list
        ↓
User taps to connect
        ↓
Phone initiates GATT connection to lock's MAC address
```

**Note:** The MAC address is looked up *after* filtering by Service UUID, not before.

---

## Connection flow (after user taps)

```
Phone (Central)                          Lock (Peripheral)
     │                                         │
     │──── Connect request ──────────────────→ │
     │                                         │
     │← ─── Connection established ──────────← │
     │                                         │
     │──── Service Discovery ────────────────→ │
     │     (enumerate all services & chars)    │
     │                                         │
     │← ─── Here are my services/chars ──────← │
     │                                         │
     │──── Characteristic Write ──────────────→ │  (send command)
     │     [TriggerLockCommand bytes]         │
     │                                         │
     │← ─── Characteristic Notify ───────────← │  (receive response)
     │     [CommandResult bytes]              │
     │                                         │
     │──── Disconnect ────────────────────────→ │
```

---

## Auto-connect patterns (connect without UI picker)

The basic flow above requires the user to tap a device from a list. But you can skip the picker by remembering the device ahead of time:

### Pattern 1: Stored local state (one-time tap, then auto-connect)
```
First time:
  User: Taps "Front Door Lock" in the picker
  Phone: Stores lock's MAC or stable ID locally
  
Subsequent times:
  App opens
  Phone: Looks up stored ID → scans for that device → auto-connects
  User: No picker shown; device connected in background
```

### Pattern 2: Server remembers user's primary device
```
First time:
  User: Taps "Front Door Lock" in picker
  Phone: Sends to server: "I'm connecting to lock physicalLockId=12345"
  Server: Stores "this user's primary lock = physicalLockId 12345"

Subsequent times:
  App opens
  Phone: Asks server: "What's my primary lock?"
  Server: "physicalLockId 12345"
  Phone: Scans, finds device with that ID, auto-connects
```

### Pattern 3: QR code + server forwarding (your scenario)
```
Setup time:
  Device sticker has QR code encoding: qrDeviceId="ABC123"
  
First connection:
  User: Scans QR code with phone camera
  Phone: Sends to server: "User scanned QR code ABC123"
  Server: Looks up: "QR ABC123 maps to physicalLockId=12345, 
                     MAC=XX:XX:XX, Service UUID=..."
  Server: Returns to phone: "Device info for lock physicalLockId=12345"
  Phone: Uses that info to scan for device by Service UUID, 
         finds it, auto-connects

Later unlocks:
  (Same as Pattern 2 from here — server remembers the device)
```

**Why Pattern 3 is useful:**
- QR code is **offline-first**: no need to type serial numbers or MAC addresses
- Server acts as a **device registry**: one QR code → lookup all device details
- Scales to multiple devices per user: scan different QR codes, server returns different devices
- **No confusion**: user doesn't pick the wrong device (server decides which physicalLockId the QR maps to)

**Design considerations for your implementation:**
1. QR code must encode a unique **qrDeviceId** or **physicalLockId**
2. Server must have a lookup table: `qrDeviceId` → `{physicalLockId, serviceUuid, macAddressIfNeeded, ...}`
3. Phone caches the device info locally (to work offline after first scan)
4. Subsequent taps on the same device use cached info + auto-scan/connect

---

## How this applies to the parent project

| Step | Tapkey pattern | Your PWA relay |
|------|---|---|
| **Scan** | Filter by Service UUID `6e65742e-7470-6b79-...` | Filter by your Service UUID (TBD) |
| **Peripheral** | BLE lock hardware | ESP32-S3 device |
| **Service UUIDs** | `net.tpky` (proprietary) | Define your own per environment |
| **Characteristics** | Opaque (inside SDK binary) | Define your own (write + notify) |
| **Command format** | TLCP (proprietary) | Define your own protocol |
| **Response format** | CommandResult enum | Define your own result codes |

**Design task:** Before building the relay, create a `GATT-CONTRACT.md` that specifies:
1. Your Service UUID (prod and sandbox)
2. Your Characteristic UUIDs (command write + response notify)
3. Your wire protocol (framing, encryption if any)
4. Your result code enum

Then both firmware and PWA relay can be built to that contract independently.

---

## Glossary

| Term | Meaning |
|------|---------|
| **BLE Central** | The device that scans and connects (phone, PWA) |
| **BLE Peripheral** | The device being scanned and connected to (lock, ESP32) |
| **Service UUID** | Stable 128-bit identifier advertised by peripheral; used to filter at scan time |
| **Characteristic UUID** | 128-bit identifier for a data point *within* a service |
| **GATT** | Generic Attribute Profile; the data model (services/characteristics) |
| **Advertisement packet** | Broadcast message periodic sent by peripheral saying "I exist, here are my services" |
| **Scan filter** | Rules the central uses to decide which advertisements to accept |
| **MAC address** | 48-bit hardware address (ephemeral in modern BLE) |
| **TLCP** | Tapkey Lock Control Protocol (proprietary wire format above GATT) |
