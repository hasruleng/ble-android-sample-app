# Reference — MessageResolver.kt

Source: `tapkey-keyring-app-template-android` (Apache-2.0, © 2022 Tapkey GmbH)
File: `app/src/main/java/io/tapkey/util/MessageResolver.kt`

This class is the most complete public reference for the full error space of a Tapkey BLE command round-trip. It is copied here as a study artifact because the parent project's PWA relay needs to handle the same error categories (different SDK, same protocol layer).

**Do not copy code.** Use the enum names and exception types as a checklist of what can go wrong.

---

## Error Architecture

`MessageResolver` has four entry points, each covering a different layer of the stack:

| Method | Input | Layer |
|--------|-------|-------|
| `getMessage(CommandResultCode)` | High-level command outcome | App → SDK boundary |
| `getMessage(Exception)` | Runtime exception type | BLE transport / SDK internals |
| `getMessage(errorCode, TkErrorDescriptor)` | Raw TLCP error string | Lock firmware / protocol |
| `getMessage(ValidityError)` | Parsed TLCP validity enum | TLCP protocol layer |

For the PWA relay: `CommandResultCode` and `ValidityError` define the observable result space at the relay boundary. Exception types (`ConnectionLostException`, `BleException`) define what can go wrong in the transport before a result is even received.

---

## CommandResultCode — App-layer outcomes

```kotlin
fun getMessage(commandResultCode: CommandResult.CommandResultCode): String {
    return when (commandResultCode) {
        CommandResult.CommandResultCode.Ok -> context.getString(R.string.success)
        CommandResult.CommandResultCode.WrongLockMode -> context.getString(R.string.wrong_lock_mode)
        CommandResult.CommandResultCode.LockVersionTooOld -> context.getString(R.string.lock_version_too_old_error)
        CommandResult.CommandResultCode.LockVersionTooYoung -> context.getString(R.string.lock_version_too_young_error)
        CommandResult.CommandResultCode.LockNotFullyAssembled -> context.getString(R.string.lock_not_fully_assembled)
        CommandResult.CommandResultCode.ServerCommunicationError -> if (isNetworkOnline()) context.getString(R.string.server_communication_error) else context.getString(R.string.server_communication_error_offline)
        CommandResult.CommandResultCode.LockDateTimeInvalid -> context.getString(R.string.lock_date_time_invalid_error)
        CommandResult.CommandResultCode.TemporarilyUnauthorized -> context.getString(R.string.unauthorized_not_yet_valid_error)
        CommandResult.CommandResultCode.Unauthorized_NotYetValid -> context.getString(R.string.unauthorized_not_yet_valid_error)
        CommandResult.CommandResultCode.Unauthorized -> context.getString(R.string.unauthorized_error)
        CommandResult.CommandResultCode.LockCommunicationError -> context.getString(R.string.lock_communication_error)
        CommandResult.CommandResultCode.UserSpecificError -> context.getString(R.string.generic_error)
        CommandResult.CommandResultCode.TechnicalError -> context.getString(R.string.generic_error)
        else -> context.getString(R.string.generic_error)
    }
}
```

### Severity classification

Each code carries a severity the UI uses to choose an icon:

```kotlin
fun getSeverity(commandResultCode: CommandResult.CommandResultCode): Severity {
    return when (commandResultCode) {
        CommandResult.CommandResultCode.Ok                    -> Severity.Log
        CommandResult.CommandResultCode.WrongLockMode         -> Severity.Warning
        CommandResult.CommandResultCode.LockVersionTooOld     -> Severity.Warning
        CommandResult.CommandResultCode.LockVersionTooYoung   -> Severity.Warning
        CommandResult.CommandResultCode.LockNotFullyAssembled -> Severity.Error
        CommandResult.CommandResultCode.ServerCommunicationError -> Severity.Error
        CommandResult.CommandResultCode.LockDateTimeInvalid   -> Severity.Warning
        CommandResult.CommandResultCode.TemporarilyUnauthorized  -> Severity.Warning
        CommandResult.CommandResultCode.Unauthorized_NotYetValid -> Severity.Warning
        CommandResult.CommandResultCode.Unauthorized          -> Severity.Warning
        CommandResult.CommandResultCode.LockCommunicationError -> Severity.Error
        CommandResult.CommandResultCode.UserSpecificError     -> Severity.Error
        CommandResult.CommandResultCode.TechnicalError        -> Severity.Error
        else -> Severity.Error
    }
}
```

**PWA relay note:** Warning-severity codes (`Unauthorized`, `LockDateTimeInvalid`, `WrongLockMode`) mean the lock received and processed the command but rejected it on policy grounds — the relay delivered successfully. Error-severity codes may indicate delivery failure. These are distinct retry/idempotency cases.

---

## Exception Types — Transport-layer failures

```kotlin
fun getMessage(exception: java.lang.Exception?): String {
    var e = exception
    if (exception is AsyncException) {
        e = exception.syncSrcException   // unwrap promise wrapper
    }

    if (e is TkException) return getMessage(e)

    if (e is ConnectionLostException) return context.getString(R.string.connection_lost)
    if (e is BleException) return context.getString(R.string.ble_connection_err)
    if (e is ServerCommunicationException) return getMessage(CommandResult.CommandResultCode.ServerCommunicationError)
    if (e is IOException) return unknownMessage()

    return unknownMessage()
}
```

| Exception | Meaning | PWA relay implication |
|-----------|---------|----------------------|
| `ConnectionLostException` | BLE link dropped mid-session | Command may or may not have reached the lock — delivery uncertain |
| `BleException` | BLE adapter / scan error | Command never sent |
| `ServerCommunicationException` | Lock couldn't reach Tapkey backend during command | Retry may succeed once connectivity restored |
| `IOException` | Generic I/O failure | Treat as unknown delivery state |

**Key distinction for idempotency design:** `ConnectionLostException` is the dangerous case — the relay lost the connection *after* sending but *before* receiving a result. The ESP32-S3 relay must handle this as "delivery unknown, may have executed."

---

## ValidityError — TLCP protocol-layer errors

These come from the lock firmware via TLCP, one layer below `CommandResultCode`.

```kotlin
fun getMessage(error: ValidityError): String {
    return when (error) {
        ValidityError.Ok                      -> context.getString(R.string.ok)
        ValidityError.NfcTransportError       -> context.getString(R.string.nfc_transport_error)
        ValidityError.ConcurrencyError        -> context.getString(R.string.concurrency_error)
        ValidityError.LockProtocolError       -> context.getString(R.string.lock_protocol_error)
        ValidityError.UnexpectedLockResponseError -> context.getString(R.string.unexpected_lock_response_error)
        ValidityError.IllegalLockState        -> context.getString(R.string.illegal_lock_state)
        ValidityError.WrongLockMode           -> context.getString(R.string.wrong_lock_mode)
        ValidityError.LockFwTooOld            -> context.getString(R.string.lock_version_too_old_error)
        ValidityError.DifferentDevice         -> context.getString(R.string.different_device)
        ValidityError.SessionBroken           -> context.getString(R.string.session_broken)
        ValidityError.NetworkRelatedError     -> if (isNetworkOnline()) context.getString(R.string.network_error) else context.getString(R.string.network_error_offline)
        ValidityError.ServerSideError         -> context.getString(R.string.server_side_error)
        ValidityError.LockCommunicationError  -> context.getString(R.string.lock_communication_error)
        ValidityError.TransportProtocolError  -> context.getString(R.string.transport_protocol_error)
        ValidityError.Generic                 -> unknownMessage()
        else -> unknownMessage()
    }
}
```

Notable codes for the PWA relay study:

- **`SessionBroken`** — the TLCP session was interrupted; maps to `ConnectionLostException` at the app layer
- **`ConcurrencyError`** — lock received concurrent requests; relevant for idempotency design
- **`DifferentDevice`** — lock rejected because command came from an unexpected device identity
- **`TransportProtocolError`** / **`LockProtocolError`** — framing errors at the GATT layer

---

## UnexpectedLockResponseError — response code extraction

```kotlin
fun getMessage(errorCode: String, tkErrorDescriptor: TkErrorDescriptor): String {
    val validityError: ValidityError? = try {
        ValidityError.valueOf(errorCode)
    } catch (ignore: IllegalArgumentException) {
        null
    }

    if (validityError == ValidityError.UnexpectedLockResponseError) {
        val responseCode = getResponseCode(tkErrorDescriptor)  // extracts "responseCode" from error details map
        if (responseCode != null) {
            val message = getMessageForResponseCode(responseCode)
            if (message != null) return message
        }
    }

    return validityError?.let { getMessage(it) } ?: unknownMessage()
}

// Only "NotFullyAssembled" is specifically handled; everything else falls through to generic
private fun getMessageForResponseCode(responseCode: String?): String? {
    return if (responseCode == null) null else when (responseCode) {
        "NotFullyAssembled" -> context.getString(R.string.lock_not_fully_assembled)
        else -> null
    }
}
```

---

## Network state check

```kotlin
fun isNetworkOnline(): Boolean {
    try {
        val cm = context.getSystemService(Context.CONNECTIVITY_SERVICE) as ConnectivityManager
        var netInfo = cm.getNetworkInfo(0)
        if (netInfo != null && netInfo.state == NetworkInfo.State.CONNECTED) return true else {
            netInfo = cm.getNetworkInfo(1)
            if (netInfo != null && netInfo.state == NetworkInfo.State.CONNECTED) return true
        }
    } catch (e: Exception) {
        Log.d(this.javaClass.name, "Can't load ConnectivityManager", e)
    }
    return false
}
```

Used to give different user-facing messages for `ServerCommunicationError` and `NetworkRelatedError` depending on whether the phone has connectivity. Not directly applicable to the PWA relay, but confirms the pattern: network-state-aware error messaging is intentional Tapkey design.
