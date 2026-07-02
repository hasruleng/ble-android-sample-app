# SC1 — App Lifecycle Entry Point

**Question:** When does the app initialize? When does BLE init happen? Where does MainActivity route the user?

**Files:** [App.java](../../app/src/main/java/net/tpky/demoapp/App.java), [MainActivity.java](../../app/src/main/java/net/tpky/demoapp/MainActivity.java)

---

## App.onCreate — Global setup

`App.java:21-44` runs once per process start. Steps in order:

1. **Line 29** — `new TapkeyServiceFactoryBuilder(this)`
2. **Line 30** — `.setTokenRefreshHandler(new SampleTokenRefreshHandler(this))` — wires the silent-refresh callback (see [SC4](sc4-token-exchange.md))
3. **Line 31** — `.build()` — stores the singleton `TapkeyServiceFactory` used by all activities/fragments
4. **Line 43** — `PollingScheduler.register(this, 1, PollingScheduler.DEFAULT_INTERVAL)` — schedules a background poll for Tapkey server notifications every 8 hours (jobId=1)

**Key finding:** **BLE is NOT initialized in `App.onCreate`.** No BLE scanner, no BLE communicator, no permissions request. The entire BLE stack is idle until a specific fragment demands it.

---

## When BLE actually starts

BLE initialization is lazy and happens in [KeyListFragment.java](../../app/src/main/java/net/tpky/demoapp/KeyListFragment.java):

- **Line 152–153** — `bleLockScanner` and `bleLockCommunicator` are retrieved from the factory in `onViewCreated`
- **Line 190** — `bleLockScanner.startForegroundScan()` is called only after runtime permissions are granted (see [SC2](sc2-device-list-ui.md) / UC4)

This means the BLE stack is dormant until the user (a) is logged in and (b) navigates to the key list. Login screens, About screen, and initial routing all execute without any BLE calls.

---

## MainActivity.onCreate — Routing

`MainActivity.java:36-88`:

```
onCreate
├── Line 42–43: get App instance + TapkeyServiceFactory
├── Line 45: userManager = factory.getUserManager()
├── Line 47–58: inflate layout, setup drawer
├── Line 66–70: if !AuthStateManager.isLoggedIn(this) → LoginActivity
├── Line 74–84: else if userManager.getUsers().isEmpty() →
│                setLoggedOut() + LoginActivity  (SDK/local desync guard)
└── Line 86: refreshUi()  (populate nav header username)
```

The double auth guard (local `SharedPreferences` **and** SDK user store) is documented in [UC2](../uc2-session-check.md).

---

## PWA relay implication

For the parent project: **defer BLE initialization until the user is authenticated and on the "trigger" screen.** No point warming up Web Bluetooth on the login screen. Match this lazy pattern — it also matches Web Bluetooth's user-gesture requirement (`navigator.bluetooth.requestDevice()` must be called from a user-initiated event).
