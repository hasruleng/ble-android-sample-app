# android-sample-app — BLE Architecture Study Reference

> **Upstream status:** Officially discontinued by Tapkey. Current official reference: [tapkey-keyring-app-template-android](https://github.com/tapkey/tapkey-keyring-app-template-android).
> This fork is a personal study reference for BLE architecture patterns that inform a PWA relay design — see [CLAUDE.md](CLAUDE.md) for full context.

---

## System Context

The app sits between a user, a backend auth stack, and a physical BLE lock. This diagram shows what crosses each system boundary.

```mermaid
C4Context
    title System Context — Tapkey Android Sample App

    Person(user, "User", "Mobile app end-user")

    System_Boundary(app, "Tapkey Sample App") {
        System(tapkeyApp, "Android App", "Manages auth, lists keys, triggers BLE locks")
    }

    System_Ext(sampleBackend, "Sample Backend", "Issues external JWTs, serves grant metadata")
    System_Ext(tapkeyAuth, "Tapkey Auth Server", "OAuth 2.0 — exchanges external JWT for Tapkey access token")
    System_Ext(bleLock, "BLE Lock", "Physical Tapkey lock, communicates via GATT/TLCP")
    System_Ext(androidOS, "Android OS", "Manages runtime BLE permissions")

    Rel(user, tapkeyApp, "Creates account, triggers locks, views keys")
    Rel(tapkeyApp, sampleBackend, "Fetches external JWT + grant metadata", "HTTPS / Basic Auth")
    Rel(tapkeyApp, tapkeyAuth, "Exchanges JWT for access token", "OAuth 2.0")
    Rel(tapkeyApp, bleLock, "Sends TriggerLock command", "BLE GATT / TLCP")
    Rel(bleLock, tapkeyApp, "Returns CommandResult", "BLE GATT / TLCP")
    Rel(androidOS, tapkeyApp, "Grants/denies BLE permissions", "Runtime permission")
```

---

## Documentation

| Document | What it covers |
|----------|---------------|
| [CLAUDE.md](CLAUDE.md) | Study goals, checklist, project context, known limitations |
| [docs/USE_CASES.md](docs/USE_CASES.md) | All 9 use cases with class + sequence diagrams |
| [docs/comparison-error-handling.md](docs/comparison-error-handling.md) | Error handling comparison vs. Tapkey App Template |
| [docs/ref-message-resolver.md](docs/ref-message-resolver.md) | Annotated `MessageResolver.kt` — full BLE error taxonomy |
