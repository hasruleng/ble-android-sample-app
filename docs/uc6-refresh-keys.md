# UC6 — Refresh Keys (Poll for Notifications)

Trigger a poll to the Tapkey backend for pending notifications (typically new/revoked grants), which in turn refreshes the local key cache and UI.

## Actors

- **User** — taps Refresh menu item (manual path)
- **App** — `MainActivity`, `App` (background scheduler)
- **Tapkey SDK** — `NotificationManager`, `KeyManager`, `PollingScheduler`

## Class Diagram

```mermaid
classDiagram
    class App {
        +onCreate()
    }
    class MainActivity {
        +onNavigationItemSelected(MenuItem)
        -refreshKeys()
    }
    class NotificationManager {
        <<Tapkey SDK>>
        +pollForNotificationsAsync(ct) Promise~Void~
    }
    class KeyManager {
        <<Tapkey SDK>>
        +getKeyUpdateObservable() Observable
    }
    class PollingScheduler {
        <<Tapkey SDK>>
        +register(context, id, interval)
    }
    class KeyListFragment {
        -onKeyUpdate(force)
    }

    App --> PollingScheduler : register background poll (8h)
    MainActivity --> NotificationManager : manual pollForNotificationsAsync
    NotificationManager ..> KeyManager : fires keyUpdate
    KeyManager --> KeyListFragment : observer callback
```

## Sequence Diagram

```mermaid
sequenceDiagram
    actor User
    participant MA as MainActivity
    participant NM as NotificationManager (SDK)
    participant TB as Tapkey Backend
    participant KM as KeyManager (SDK)
    participant KLF as KeyListFragment

    alt Manual refresh
        User->>MA: nav drawer → Refresh
        MA->>NM: pollForNotificationsAsync(ct)
    else Background poll (App.onCreate)
        Note over MA,NM: PollingScheduler fires every 8h
        NM->>NM: scheduled call
    end
    NM->>TB: fetch pending notifications
    TB-->>NM: notifications
    NM-->>MA: complete (or catchOnUi log)
    NM->>KM: apply key/grant updates
    KM->>KLF: keyUpdate observable fires
    KLF->>KLF: onKeyUpdate(false) — see UC3
```

## Explanation

1. **Two triggers, one flow:**
    - **Manual** — user taps Refresh in the nav drawer → `MainActivity.refreshKeys` → `pollForNotificationsAsync`
    - **Background** — `App.onCreate` registers a `PollingScheduler` at 8-hour intervals (`PollingScheduler.DEFAULT_INTERVAL`) so keys stay reasonably fresh without user action
2. **Effect** — Notifications from the Tapkey backend are applied to the SDK's local key store. Any change fires the `KeyManager` update observable, which `KeyListFragment` is already listening to (UC3), triggering `onKeyUpdate(false)` and a fresh grant fetch.
3. **Silent errors** — `pollForNotificationsAsync` errors are swallowed with a log; the user sees no direct feedback (Refresh menu simply completes).

## Error Paths

| Failure | Handling |
|---------|----------|
| Network error during poll | `catchOnUi` logs `"Error while polling for notifications."` |
| Any exception | `finallyOnUi` logs completion; no UI notification |

## Files

- [app/src/main/java/net/tpky/demoapp/App.java](../app/src/main/java/net/tpky/demoapp/App.java) (line ~43)
- [app/src/main/java/net/tpky/demoapp/MainActivity.java](../app/src/main/java/net/tpky/demoapp/MainActivity.java) (lines ~124–139)
