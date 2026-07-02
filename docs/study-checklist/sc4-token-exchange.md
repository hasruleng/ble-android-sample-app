# SC4 — Token Exchange Dance

**Question:** How does the app get and refresh the Tapkey access token? What is the flow shape (ignoring SDK specifics)?

**Files:** [AuthStateManager.java](../../app/src/main/java/net/tpky/demoapp/AuthStateManager.java), [SampleTokenRefreshHandler.java](../../app/src/main/java/net/tpky/demoapp/SampleTokenRefreshHandler.java), [TapkeyTokenExchangeManager.java](../../app/src/main/java/net/tpky/demoapp/TapkeyTokenExchangeManager.java)

---

## Credential persistence

`AuthStateManager.java` stores two fields in `PreferenceManager.getDefaultSharedPreferences()`:

| Key | Type | Purpose |
|-----|------|---------|
| `KEY_USERNAME` | String | Basic Auth to Sample Backend + input to refresh flow |
| `KEY_PASSWORD` | String | Same. **Plaintext.** |

API surface: `setLoggedIn(ctx, u, p)`, `setLoggedOut(ctx)`, `isLoggedIn(ctx)`.

**Security note:** plaintext password persistence is a deliberate tradeoff to enable silent token refresh without user re-prompt. The parent project (`Secure-datasharing-platform`) does not need this — its tokens are single-use, session-scoped, and the phone is treated as untrusted.

---

## The three-actor dance

Every access token acquisition (initial login and silent refresh) follows the same shape:

```
┌────────────┐   1. Basic Auth        ┌─────────────────┐
│    App     │───────────────────────►│  Sample Backend │
│            │◄─── externalToken (JWT)│                 │
└──────┬─────┘                        └─────────────────┘
       │
       │   2. OAuth token exchange     ┌──────────────────┐
       └──────────────────────────────►│ Tapkey Auth      │
                                       │ Server (OIDC)    │
              ◄──────── accessToken ───│                  │
                                       └──────────────────┘

       │   3. logInAsync(accessToken)  ┌──────────────────┐
       └──────────────────────────────►│  Tapkey SDK      │
                                       │  (UserManager)   │
                                       └──────────────────┘
```

- Step 1 uses the persisted username/password (Basic Auth).
- Step 2 is the interesting bit — a **custom OAuth token exchange grant**, not a standard flow.
- Step 3 hands the resulting Tapkey access token to the SDK for use in server calls and BLE flows.

---

## OAuth token exchange details

`TapkeyTokenExchangeManager.exchangeToken()` at `TapkeyTokenExchangeManager.java:52-84`:

1. **OIDC discovery** — `AuthorizationServiceConfiguration.fetchFromIssuer(<tapkey_authorization_server>)` to get token endpoint URL
2. **Build TokenRequest** with:
   - `grantType`: `http://tapkey.net/oauth/token_exchange` (custom, non-standard)
   - `scopes`: `register:mobiles`, `read:user`, `handle:keys`
   - `additionalParameters`:
     - `provider`: `tapkey_identity_provider_id` (from resources)
     - `subject_token_type`: `jwt`
     - `subject_token`: the externalToken from step 1
     - `audience`: `tapkey_api`
     - `requested_token_type`: `access_token`
3. **Perform request** — `authService.performTokenRequest(...)` returns `TokenResponse` containing `accessToken`

This is a variant of [RFC 8693](https://datatracker.ietf.org/doc/html/rfc8693) (OAuth 2.0 Token Exchange) with a custom grant URI. Standard-ish, but not portable.

---

## Silent refresh

`SampleTokenRefreshHandler.refreshAuthenticationAsync()` at `SampleTokenRefreshHandler.java:28-43`:

Invoked automatically by the Tapkey SDK when it detects an expired access token during any operation (list keys, trigger lock, poll notifications — see [UC8](../uc8-token-refresh.md)).

```java
if (!AuthStateManager.isLoggedIn(context))
    throw new TkException(TokenRefreshFailed);

return sampleServerManager.getExternalToken(u, p)          // step 1 again
    .continueAsyncOnUi(tokenExchangeManager::exchangeToken) // step 2 again
    .continueOnUi(response -> response.accessToken)
    .catchOnUi(ex -> { throw new TkException(TokenRefreshFailed); });
```

It re-runs steps 1 + 2 of the dance silently. Step 3 (SDK login) is not repeated — the SDK just gets the new token to attach to pending requests.

On unrecoverable failure, `onRefreshFailed` starts `LoginActivity`, forcing the user through the full flow again.

---

## PWA relay implication

The three-actor shape is worth borrowing:
- **App backend** issues an app-scoped credential (Basic-Auth JWT here, could be a session cookie)
- **Identity service** exchanges that for a **device-scoped** token that only the physical device recognises
- **On-device SDK / relay code** attaches the device-scoped token to actual command traffic

For the parent project this maps to: user session → server signs a one-time BLE token → PWA relays that token to the ESP32-S3.

**Do NOT copy:**
- The token exchange grant URI — it's Tapkey-proprietary
- Plaintext password persistence — irrelevant for one-time tokens
- OIDC discovery — the parent project's token endpoint is fixed and known

**Do copy:**
- Separation between app-backend auth and device-facing token — do not let the phone hold the credential the device trusts
- Silent-refresh handler shape — a callback the SDK/relay can invoke when the current token stops working
