# UC7 — Log Out

Clear local credentials, log out of the Tapkey SDK, and return to `LoginActivity`.

## Actors

- **User** — taps Sign Out
- **App** — `MainActivity`, `AuthStateManager`
- **Tapkey SDK** — `UserManager.logOutAsync`

## Class Diagram

```mermaid
classDiagram
    class MainActivity {
        +onNavigationItemSelected(MenuItem)
        -logOut()
    }
    class AuthStateManager {
        +setLoggedOut(context)
    }
    class UserManager {
        <<Tapkey SDK>>
        +logOutAsync(userId, ct) Promise~Void~
    }
    class LoginActivity

    MainActivity --> UserManager : logOutAsync
    MainActivity --> AuthStateManager : setLoggedOut
    MainActivity --> LoginActivity : startActivityForResult
```

## Sequence Diagram

```mermaid
sequenceDiagram
    actor User
    participant MA as MainActivity
    participant UM as UserManager (SDK)
    participant ASM as AuthStateManager
    participant LA as LoginActivity

    User->>MA: nav drawer → Sign Out
    alt SDK has a user
        MA->>UM: logOutAsync(userId)
        UM-->>MA: complete (or logged error)
    end
    MA->>ASM: setLoggedOut(ctx)
    MA->>LA: startActivityForResult(LOGON_REQUEST_CODE)
```

## Explanation

1. **SDK logout** — If a Tapkey user is present, `UserManager.logOutAsync` is called. This can fail but is treated as non-fatal.
2. **Local logout is unconditional** — `AuthStateManager.setLoggedOut(this)` always runs, clearing `SharedPreferences`. This ensures the user is fully logged out locally even if the SDK call fails.
3. **Redirect** — `LoginActivity` is started with the `LOGON_REQUEST_CODE`, replacing `MainActivity` in the back stack.

## Error Paths

| Failure | Handling |
|---------|----------|
| SDK `logOutAsync` throws | Logged as `"Could not log out user: ..."`; does not block local logout or redirect |

## Files

- [app/src/main/java/net/tpky/demoapp/MainActivity.java](../app/src/main/java/net/tpky/demoapp/MainActivity.java) (lines ~120, ~154)
- [app/src/main/java/net/tpky/demoapp/AuthStateManager.java](../app/src/main/java/net/tpky/demoapp/AuthStateManager.java)
