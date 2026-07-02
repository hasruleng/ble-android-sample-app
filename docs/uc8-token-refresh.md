# UC8 — Automatic Tapkey Token Refresh

When the Tapkey SDK detects an expired access token during an operation, it invokes the app-registered refresh handler to silently obtain a new one — using the credentials persisted at login.

## Actors

- **App** — `App`, `SampleTokenRefreshHandler`, `SampleServerManager`, `TapkeyTokenExchangeManager`, `AuthStateManager`
- **Sample Backend** — `GET /user/tapkey-token`
- **Tapkey Auth Server** — OAuth token exchange
- **Tapkey SDK** — invokes handler transparently

## Class Diagram

```mermaid
classDiagram
    class App {
        +onCreate()
    }
    class TapkeyServiceFactoryBuilder {
        <<Tapkey SDK>>
        +setTokenRefreshHandler(handler)
    }
    class SampleTokenRefreshHandler {
        +refreshAuthenticationAsync(userId, ct) Promise~String~
        +onRefreshFailed(userId)
    }
    class AuthStateManager {
        +isLoggedIn(context) boolean
        +getUsername(context)
        +getPassword(context)
    }
    class SampleServerManager {
        +getExternalToken(u, p) Promise~String~
    }
    class TapkeyTokenExchangeManager {
        +exchangeToken(externalToken) Promise~String~
    }

    App --> TapkeyServiceFactoryBuilder : setTokenRefreshHandler
    TapkeyServiceFactoryBuilder --> SampleTokenRefreshHandler
    SampleTokenRefreshHandler --> AuthStateManager
    SampleTokenRefreshHandler --> SampleServerManager
    SampleTokenRefreshHandler --> TapkeyTokenExchangeManager
```

## Sequence Diagram

```mermaid
sequenceDiagram
    participant SDK as Tapkey SDK
    participant H as SampleTokenRefreshHandler
    participant ASM as AuthStateManager
    participant SSM as SampleServerManager
    participant SB as Sample Backend
    participant TEM as TapkeyTokenExchangeManager
    participant TAS as Tapkey Auth Server
    participant LA as LoginActivity

    Note over SDK: access token expired during any op<br/>(list keys, trigger lock, poll)
    SDK->>H: refreshAuthenticationAsync(userId, ct)
    H->>ASM: isLoggedIn(ctx)?
    alt not logged in
        ASM-->>H: false
        H-->>SDK: throw TkException(TokenRefreshFailed)
        SDK->>H: onRefreshFailed(userId)
        H->>LA: startActivity(LoginActivity)
    else logged in
        ASM-->>H: true (u, p)
        H->>SSM: getExternalToken(u, p)
        SSM->>SB: GET /user/tapkey-token (Basic Auth)
        SB-->>SSM: externalToken (JWT)
        SSM-->>H: externalToken
        H->>TEM: exchangeToken(externalToken)
        TEM->>TAS: OAuth token exchange
        TAS-->>TEM: accessToken
        TEM-->>H: accessToken
        H-->>SDK: accessToken
        Note over SDK: original operation continues transparently
    end
```

## Explanation

1. **Registration** — At app startup, `App.onCreate` wires the handler via `TapkeyServiceFactoryBuilder.setTokenRefreshHandler(new SampleTokenRefreshHandler(this))`. From this point on, any expired token triggers the callback.
2. **Silent re-auth** — The handler mirrors the login flow (UC1) but without user interaction: fetch external token from Sample Backend (Basic Auth using stored username/password) → exchange for Tapkey access token → return.
3. **This is why passwords are persisted** — `AuthStateManager` stores credentials in `SharedPreferences` specifically so silent refresh works without prompting the user. This is a deliberate tradeoff (persisted password vs. UX) noted in the parent project's threat model.
4. **Failure escalation** — If refresh cannot complete (not logged in locally, or any exception), the handler throws `TkException(TokenRefreshFailed)`. The SDK then calls `onRefreshFailed`, which sends the user to `LoginActivity` to re-authenticate.

## Error Paths

| Cause | Handling |
|-------|----------|
| Not logged in locally | Immediate `TkException(TokenRefreshFailed)` |
| Backend down / bad creds (e.g., password rotated) | `catchOnUi` → `TkException(TokenRefreshFailed)` |
| SDK receives failure → calls `onRefreshFailed` | Handler starts `LoginActivity` |

## Files

- [app/src/main/java/net/tpky/demoapp/App.java](../app/src/main/java/net/tpky/demoapp/App.java)
- [app/src/main/java/net/tpky/demoapp/SampleTokenRefreshHandler.java](../app/src/main/java/net/tpky/demoapp/SampleTokenRefreshHandler.java)
- [app/src/main/java/net/tpky/demoapp/SampleServerManager.java](../app/src/main/java/net/tpky/demoapp/SampleServerManager.java)
- [app/src/main/java/net/tpky/demoapp/TapkeyTokenExchangeManager.java](../app/src/main/java/net/tpky/demoapp/TapkeyTokenExchangeManager.java)
