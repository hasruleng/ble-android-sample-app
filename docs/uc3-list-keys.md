# UC3 — List Available Keys

Show the user their keys with grant metadata and live BLE proximity indicators.

## Actors

- **User** — passive viewer
- **App** — `KeyListFragment`, `KeyItemAdapter`, `SampleServerManager`
- **Tapkey SDK** — `KeyManager`, `BleLockScanner`
- **Sample Backend** — `GET /user/grants?grantIds=...`

## Class Diagram

```mermaid
classDiagram
    class KeyListFragment {
        -KeyItemAdapter adapter
        -KeyManager keyManager
        -BleLockScanner bleLockScanner
        +onViewCreated(View, Bundle)
        +onResume()
        -onKeyUpdate(boolean forceUpdate)
    }
    class KeyItemAdapter {
        -Handler handler
        +getView(pos, convertView, parent) View
    }
    class KeyItemAdapterHandler {
        <<interface>>
        +isLockNearby(physicalLockId) boolean
        +triggerLock(physicalLockId, ct) Promise~Boolean~
    }
    class KeyManager {
        <<Tapkey SDK>>
        +queryLocalKeysAsync(userId, ct) Promise~List~KeyDetails~~
        +getKeyUpdateObservable() Observable
    }
    class BleLockScanner {
        <<Tapkey SDK>>
        +startForegroundScan()
        +getLocksChangedObservable() Observable
        +isLockNearby(id) boolean
    }
    class SampleServerManager {
        +getGrants(username, password, grantIds[]) Promise~List~ApplicationGrantDto~~
    }
    class ApplicationGrantDto {
        +String id
        +String title
        +String subtitle
    }

    KeyListFragment --> KeyItemAdapter : populates
    KeyListFragment --> KeyManager
    KeyListFragment --> BleLockScanner
    KeyListFragment --> SampleServerManager
    KeyItemAdapter ..> KeyItemAdapterHandler : delegate
    KeyListFragment ..|> KeyItemAdapterHandler : implements
    SampleServerManager --> ApplicationGrantDto : returns
```

## Sequence Diagram

```mermaid
sequenceDiagram
    actor User
    participant KLF as KeyListFragment
    participant KM as KeyManager (SDK)
    participant SSM as SampleServerManager
    participant SB as Sample Backend
    participant BLS as BleLockScanner (SDK)
    participant Adapter as KeyItemAdapter

    User->>KLF: opens Key List
    KLF->>KM: getKeyUpdateObservable().addObserver
    KLF->>BLS: getLocksChangedObservable().addObserver
    KLF->>BLS: startForegroundScan()
    KLF->>KLF: onKeyUpdate(false)
    KLF->>KM: queryLocalKeysAsync(userId)
    KM-->>KLF: List~KeyDetails~
    KLF->>SSM: getGrants(u, p, grantIds[])
    SSM->>SB: GET /user/grants?grantIds=... (Basic Auth)
    SB-->>SSM: [ApplicationGrantDto,...]
    SSM-->>KLF: grants
    KLF->>Adapter: clear() + addAll(zip(keys, grants))
    Adapter-->>User: rendered list

    loop BLE proximity updates
        BLS-->>KLF: locksChanged event
        KLF->>Adapter: notifyDataSetChanged()
        Adapter->>BLS: isLockNearby(physicalLockId)
        BLS-->>Adapter: true/false
        Note over Adapter: Toggle Trigger button visibility
    end

    loop Server key updates
        KM-->>KLF: keyUpdate event
        KLF->>KLF: onKeyUpdate(true)
    end
```

## Explanation

1. **Two observers** are registered in `onResume`: one for server-side key changes (`KeyManager`), one for BLE proximity changes (`BleLockScanner`). They both feed into the same adapter but for different reasons.
2. **Local key query** — `queryLocalKeysAsync` returns keys already cached on-device. Server-side notifications (UC6) refresh this cache.
3. **Grant metadata** — Each `KeyDetails` carries a `grantId`. The Sample Backend enriches these with display-friendly titles/subtitles via `getGrants`, returning `ApplicationGrantDto`s.
4. **Zipping** — Fragment joins `KeyDetails` and `ApplicationGrantDto` by `grantId` into tuples pushed to `KeyItemAdapter`.
5. **BLE proximity** — `KeyItemAdapter.getView` calls back into `handler.isLockNearby(physicalLockId)` for every visible row. When BLE scanner fires, the fragment calls `notifyDataSetChanged`; only rows whose lock is currently in range show the Trigger button.

## Error Paths

| Step | Failure | Handling |
|------|---------|----------|
| `queryLocalKeysAsync` | Any | `catchOnUi` logs, list not updated |
| `getGrants` | HTTP failure | Same |
| No SDK user | Returns early | Logged only |
| `startForegroundScan` | Missing permission | Wrapped in try/catch; UC4 handles permissions |

## Files

- [app/src/main/java/net/tpky/demoapp/KeyListFragment.java](../app/src/main/java/net/tpky/demoapp/KeyListFragment.java)
- [app/src/main/java/net/tpky/demoapp/KeyItemAdapter.java](../app/src/main/java/net/tpky/demoapp/KeyItemAdapter.java)
- [app/src/main/java/net/tpky/demoapp/SampleServerManager.java](../app/src/main/java/net/tpky/demoapp/SampleServerManager.java)
- [app/src/main/java/net/tpky/demoapp/ApplicationGrantDto.java](../app/src/main/java/net/tpky/demoapp/ApplicationGrantDto.java)
- Layouts: [content_main.xml](../app/src/main/res/layout/content_main.xml), [key_item.xml](../app/src/main/res/layout/key_item.xml)
