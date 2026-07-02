# SC3 — Server Credential Fetch Before BLE

**Question:** When does the app ask the server for permission before BLE? What is the fetch pattern?

**File:** [SampleServerManager.java](../../app/src/main/java/net/tpky/demoapp/SampleServerManager.java)

---

## HTTP endpoints exposed

| # | Method | Path | Auth | Payload | Response |
|---|--------|------|------|---------|----------|
| 1 | POST | `/user` | None | `{username, password, firstName, lastName}` | `{id}` |
| 2 | GET | `/user/tapkey-token` | HTTP Basic | — | `{externalToken}` (JWT) |
| 3 | GET | `/user/grants?grantIds=<csv>` | HTTP Basic | — | `[ApplicationGrantDto, ...]` |

All three go through Volley's `RequestQueue` (`SampleServerManager.java:41-172`), wrapped as `Promise<T>` via `PromiseSource`.

---

## Order of operations vs. BLE

The critical question — **what runs before any BLE call?**

```
1. LoginActivity → POST /user            (registration only)
                → GET /user/tapkey-token (fetches externalToken)
                → TapkeyTokenExchangeManager (see SC4)
                → UserManager.logInAsync(accessToken)
                → MainActivity

2. MainActivity → KeyListFragment.onResume()
                → GET /user/grants        ← still no BLE
                → adapter populated
                → bleLockScanner.startForegroundScan()  ← BLE STARTS HERE

3. User taps Trigger on a nearby row
                → bleLockCommunicator.executeCommandAsync  ← BLE COMMAND
```

**Key finding:** every server credential exchange (`/user`, `/user/tapkey-token`, `/user/grants`, plus the OAuth exchange in SC4) completes **before** any BLE operation. The app is fully authenticated and has all grant metadata cached locally before it ever scans the airwaves.

This matches the Tapkey security model: **the phone is authorised in the cloud, then acts as a courier to the lock.** The lock only sees credentials that the cloud has already blessed.

---

## Basic Auth surface

Endpoints 2 and 3 use HTTP Basic Auth with the credentials stored by `AuthStateManager` (see [SC4](sc4-token-exchange.md)). This is why the sample persists **plaintext username + password** in `SharedPreferences` — Basic Auth needs both fields on every call, not a token.

The token-exchange dance (SC4) is a separate path used only against the Tapkey Auth Server, not against the Sample Backend.

---

## PWA relay implication

The pattern **fetch cloud-signed credential → hand it to the device → device verifies offline** is exactly the parent project's design (`Secure-datasharing-platform` — server-signed one-time tokens, ESP32-S3 verifies offline).

**Directly reusable:**
- Timing: fetch all grant/credential material before scanning
- Two-stage auth: Basic-Auth to the app backend, token exchange to a separate identity service — good separation of concerns
- Local caching: don't hit the server on every trigger; cache grant metadata

**Not reusable:**
- Basic Auth with plaintext password persistence — the parent project uses ephemeral session tokens, no need to persist password
- Custom OAuth grant type `http://tapkey.net/oauth/token_exchange` — parent uses direct token issuance, no exchange step
