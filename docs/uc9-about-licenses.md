# UC9 — View About / Third-Party Licenses

Show the app version and, on demand, the bundled third-party license text.

## Actors

- **User** — opens About and optionally taps the license button
- **App** — `MainActivity`, `AboutActivity`
- **Android OS** — asset loader

## Class Diagram

```mermaid
classDiagram
    class MainActivity {
        +onNavigationItemSelected(MenuItem)
    }
    class AboutActivity {
        +onCreate(Bundle)
        +onClickShowThirdPartyLicences(View)
    }
    class BuildConfig {
        +VERSION_NAME
        +VERSION_CODE
    }
    class AssetManager {
        <<Android>>
        +open("third_party_licenses.txt") InputStream
    }
    class AlertDialog

    MainActivity --> AboutActivity : startActivity
    AboutActivity --> BuildConfig : read version
    AboutActivity --> AssetManager : read licenses asset
    AboutActivity --> AlertDialog : show
```

## Sequence Diagram

```mermaid
sequenceDiagram
    actor User
    participant MA as MainActivity
    participant AA as AboutActivity
    participant BC as BuildConfig
    participant AM as AssetManager
    participant AD as AlertDialog

    User->>MA: nav drawer → About
    MA->>AA: startActivity(AboutActivity)
    AA->>BC: VERSION_NAME, VERSION_CODE
    BC-->>AA: values
    AA-->>User: render version

    opt user taps Third-Party Licenses
        User->>AA: onClickShowThirdPartyLicences(View)
        AA->>AM: open("third_party_licenses.txt")
        alt read ok
            AM-->>AA: text content
            AA->>AD: show(text)
            AD-->>User: dialog with license text
        else IOException
            AM-->>AA: throw
            AA->>AA: log "Failed to read third party licences"
            Note over AA,User: silent to user
        end
    end
```

## Explanation

1. **About screen** — Reads `BuildConfig.VERSION_NAME` and `BuildConfig.VERSION_CODE` at `onCreate` and renders them.
2. **License dialog** — On button click, opens `third_party_licenses.txt` from the app's `assets/`, reads it, and shows it in an `AlertDialog`. The file is bundled at build time (see `third_party_licenses.txt` at the project root).
3. **Silent failure** — Any `IOException` reading the asset is logged (`"Failed to read third party licences"`) but not surfaced to the user. Given the asset is bundled, this is essentially unreachable in practice.

## Error Paths

| Cause | Handling |
|-------|----------|
| `IOException` on asset read | Logged only; no dialog shown |

## Files

- [app/src/main/java/net/tpky/demoapp/AboutActivity.java](../app/src/main/java/net/tpky/demoapp/AboutActivity.java)
- [app/src/main/java/net/tpky/demoapp/MainActivity.java](../app/src/main/java/net/tpky/demoapp/MainActivity.java)
- Layout: [app/src/main/res/layout/activity_about.xml](../app/src/main/res/layout/activity_about.xml)
- Asset: [third_party_licenses.txt](../third_party_licenses.txt)
