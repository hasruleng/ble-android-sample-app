# android-sample-app — BLE Architecture Study Reference

> **Upstream status:** This sample is officially discontinued by Tapkey. The current official reference is [tapkey-keyring-app-template-android](https://github.com/tapkey/tapkey-keyring-app-template-android).
> This fork is maintained as a personal study reference for BLE architecture patterns — see [CLAUDE.md](CLAUDE.md) for context.

Reference documentation for the Tapkey Android SDK sample app. Each use case captures the actors, class relationships, and interaction sequence to support the parent project's Phase 1 BLE architecture study.

Diagrams are written in [Mermaid](https://mermaid.js.org/) and render natively in GitHub, VS Code (with the Mermaid extension), and most Markdown viewers.

---

## Summary Use Case Diagram

All actors, external systems, use cases, and what each actor gains from the interaction.

```mermaid
graph LR
    %% Actors
    User(["👤 User"])
    AndroidOS(["⚙️ Android OS"])

    %% External Systems
    SampleBackend[["Sample Backend\n(REST API)"]]
    TapkeyAuthServer[["Tapkey Auth Server\n(OAuth 2.0)"]]
    BLELock[["BLE Lock\n(Physical Device)"]]

    %% App (system under study)
    subgraph App ["📱 Tapkey Sample App"]
        UC1["UC1 · Register / Log In"]
        UC2["UC2 · Session Check"]
        UC3["UC3 · List Available Keys"]
        UC4["UC4 · Grant BLE Permissions"]
        UC5["UC5 · Trigger Lock"]
        UC6["UC6 · Refresh Keys"]
        UC7["UC7 · Log Out"]
        UC8["UC8 · Auto Token Refresh"]
        UC9["UC9 · View About / Licenses"]
    end

    %% User interactions
    User -->|"creates account,\nenters credentials"| UC1
    User -->|"app launch"| UC2
    User -->|"views key list"| UC3
    User -->|"accepts/denies\npermission dialog"| UC4
    User -->|"taps Trigger button"| UC5
    User -->|"taps Refresh"| UC6
    User -->|"taps Sign Out"| UC7
    User -->|"opens About screen"| UC9

    %% Android OS
    AndroidOS -->|"runtime permission\nresult"| UC4

    %% Sample Backend interactions
    UC1 -->|"POST /user\nGET /user/tapkey-token"| SampleBackend
    UC3 -->|"GET /user/grants"| SampleBackend
    UC8 -->|"GET /user/tapkey-token\n(silent refresh)"| SampleBackend

    %% Tapkey Auth Server
    UC1 -->|"OAuth token exchange\n(external JWT → access token)"| TapkeyAuthServer
    UC8 -->|"OAuth token exchange\n(silent refresh)"| TapkeyAuthServer

    %% BLE Lock
    UC5 -->|"TLCP command\nover BLE GATT"| BLELock
    BLELock -->|"CommandResult\n(Ok / non-Ok)"| UC5

    %% What actors GAIN
    User -.->|"✅ gains: access to\nphysical locks"| UC5
    User -.->|"✅ gains: up-to-date\ngrant list"| UC3
    SampleBackend -.->|"✅ gains: authenticated\nuser session"| UC1
    TapkeyAuthServer -.->|"✅ gains: verified\nexternal identity"| UC1
    BLELock -.->|"✅ gains: authorised\ntrigger command"| UC5
```

---

## Comparison Report

→ [docs/comparison-error-handling.md](docs/comparison-error-handling.md) — Error handling & BLE error coverage: this repo vs. the newer Tapkey App Template

## External Reference (annotated)

→ [docs/ref-message-resolver.md](docs/ref-message-resolver.md) — Annotated copy of `MessageResolver.kt` from the Tapkey App Template: full `CommandResultCode`, `ValidityError`, and exception-type mapping with PWA relay study notes

---

## Use Case Index

| # | Use Case | File |
|---|----------|------|
| UC1 | Register Account & Log In | [docs/uc1-login-registration.md](docs/uc1-login-registration.md) |
| UC2 | Session Check on App Launch | [docs/uc2-session-check.md](docs/uc2-session-check.md) |
| UC3 | List Available Keys | [docs/uc3-list-keys.md](docs/uc3-list-keys.md) |
| UC4 | Grant Runtime BLE Permissions | [docs/uc4-ble-permissions.md](docs/uc4-ble-permissions.md) |
| UC5 | Trigger Lock (Unlock via BLE) | [docs/uc5-trigger-lock.md](docs/uc5-trigger-lock.md) |
| UC6 | Refresh Keys (Poll for Notifications) | [docs/uc6-refresh-keys.md](docs/uc6-refresh-keys.md) |
| UC7 | Log Out | [docs/uc7-logout.md](docs/uc7-logout.md) |
| UC8 | Automatic Tapkey Token Refresh | [docs/uc8-token-refresh.md](docs/uc8-token-refresh.md) |
| UC9 | View About / Third-Party Licenses | [docs/uc9-about-licenses.md](docs/uc9-about-licenses.md) |

---

## Actor Glossary

| Actor | Meaning |
|-------|---------|
| **User** | Human end-user of the mobile app |
| **App** | The Android app (activities, fragments, managers under `net.tpky.demoapp`) |
| **Sample Backend** | Application-specific server issuing external JWTs and grant metadata |
| **Tapkey Auth Server** | OAuth 2.0 server that exchanges external JWTs for Tapkey access tokens |
| **Tapkey SDK** | On-device SDK exposing `UserManager`, `KeyManager`, `BleLockScanner`, `BleLockCommunicator`, `CommandExecutionFacade`, `NotificationManager` |
| **BLE Lock** | Physical Tapkey lock reachable via BLE GATT (SDK-opaque) |

## Cross-Cutting Notes

- **Auth persistence:** `AuthStateManager` stores username/password in `SharedPreferences` (plaintext) — used for Basic Auth to the Sample Backend *and* to power silent token refresh (UC8).
- **BLE is SDK-opaque:** GATT service UUIDs, characteristic reads/writes, and TLCP framing are hidden inside `BleLockCommunicator` and `CommandExecutionFacade`. This is the primary study limitation flagged for the parent project's PWA relay design.
- **Two async event streams** feed the key list: `KeyManager.getKeyUpdateObservable` (server-side grant changes) and `BleLockScanner.getLocksChangedObservable` (BLE proximity changes). They merge inside `KeyListFragment`.
