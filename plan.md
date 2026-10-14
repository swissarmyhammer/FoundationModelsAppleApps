# Plan: FoundationModelsAppleApps — operation tools for the Apple apps

This document uses ASD-STE100 Simplified Technical English.

This package gives one FoundationModels tool for each of nine Apple apps. Each
tool is an `OperationTool` from the `Operations` library of
FoundationModelsExtras. The model selects an operation with the `op`
parameter, for example `list events`. A host can give the tools directly to a
`LanguageModelSession`. The usual host is FoundationModelsMultitool: it mounts
each tool as a group, and it changes each operation into one verb, for example
`tools.calendar.listEvents`.

The design comes from the Apple apps of Osaurus
(<https://github.com/osaurus-ai/osaurus/blob/main/docs/APPLE_APPS.md>).

**Target: macOS 27, on-device, Swift 6 strict concurrency.**

## Status of this document

This is the plan before the work starts. When the code or the README and this
document do not agree, the code and the README are correct.

## Contents

1. Goal
2. Principles
3. Source material
4. The tools and their operations
5. Shared rules
6. Backends
7. Public API
8. Package layout
9. Package.swift
10. Mount in Multitool and in the agent
11. Permissions (TCC)
12. Testing
13. Work items
14. Decisions
15. Open questions
16. Out of scope
17. References

## 1. Goal

A host mounts the tools of the apps that it wants:

```swift
let tools = try AppleAppsTools.make(apps: [.calendar, .reminders, .mail])
let session = LanguageModelSession(tools: tools, instructions: instructions)
```

In code mode, Multitool mounts each tool as a group. A model then writes:

```js
const today = await tools.calendar.listEvents({ start: "2026-10-14", end: "2026-10-14" });
const open = await tools.reminders.listReminders({ list: "Work", status: "incomplete" });
await tools.reminders.createReminder({
  list: "Work",
  title: `Prepare for ${today[0].title}`,
  due: today[0].start,
});
```

## 2. Principles

1. **One tool for each app.** Each app has one `OperationTool`. The name of the
   tool is the name of the app: `calendar`, `reminders`, `contacts`, `notes`,
   `mail`, `messages`, `maps`, `music`, `shortcuts`. Each tool has its own
   verbs and nouns.
2. **The CodeContext pattern.** Each operation is an `@Generable @Operation`
   struct. Its `execute(in:)` function is a thin shim over one backend call.
   The tool types and the operation types are `internal`. The host needs no
   `import Operations`.
3. **No dependency on Multitool.** The library links only `Operations` and
   `FoundationModelsExtras`. Multitool mounts the tools through
   `OperationDescribing`, which `OperationTool` conforms to. Only the
   integration tests link Multitool.
4. **Multitool is the main host.** The design of each op string is for the
   camelCase verb that Multitool makes from it. `list events` becomes
   `listEvents`. Each noun and each verb is a legal TypeScript identifier after
   this change.
5. **Corrections, not throws.** An error that the model can correct comes back
   as `ToolOutcome.corrective(String)`. A thrown error stops the turn. Thus an
   operation throws only for an error that the model cannot correct.
6. **No permission gate in the package.** All operations run when the host
   mounts the tool. The host selects the apps. This agrees with the posture of
   the agent: "the sandbox is the only gate".
7. **Backends behind protocols.** Each app has a backend protocol. The live
   backend uses the Apple framework, AppleScript, SQLite or a CLI. The unit
   tests use a fake backend. No unit test touches the real data of the user.
8. **All nine apps in one phase.** The work items are one flat list, in the
   order of their dependencies.

## 3. Source material

| Area | Osaurus | Our approach |
|---|---|---|
| Calendar | EventKit | EventKit (`EKEventStore`) |
| Reminders | EventKit | EventKit (`EKEventStore`) |
| Contacts | Contacts.framework | Contacts.framework (`CNContactStore`) |
| Notes | AppleScript | AppleScript through `/usr/bin/osascript` |
| Mail | AppleScript | AppleScript through `/usr/bin/osascript` |
| Messages | SQLite reads of `chat.db`, AppleScript sends | GRDB read-only reads of `chat.db`, AppleScript sends |
| Maps and location | CoreLocation, maps backend not named | CoreLocation for the location, MapKit for search, geocode and directions |
| Music | AppleScript | AppleScript through `/usr/bin/osascript` |
| Shortcuts | `/usr/bin/shortcuts` | `/usr/bin/shortcuts` through `swift-subprocess` |

We copy these rules from Osaurus:

- Dates are ISO 8601. A bare date is valid. An invalid date gives a
  correction. A date does not roll over (`2026-02-30` is not valid).
- The `end` of an all-day event is inclusive.
- A JSON `null` clears a field on update. An absent field does not change.
- An update or a delete of a recurring event needs `occurrenceStart`. `span` is
  `this_event` (the default) or `future_events`.
- A calendar query has a maximum range of four years. The result tells when the
  range was clamped (`endClamped`).
- Alarms are clamped to 0 to 4 weeks before the event.
- `append note` refuses a note that has attachments.
- Mailbox paths are `Account/Folder/Subfolder`. `\/` is a literal slash.
- `send message` verifies the delivery in `chat.db`. It does not send again
  when the result is not known.
- `search places` with `near` sorts by distance. Coordinates are range-checked.
- `play item` changes shuffle only when the call gives `shuffle`.
- The output of `run shortcut` has a maximum of 1 MiB. The result then has
  `truncated: true`. A cancel sends SIGTERM, then SIGKILL.

We do not copy these from Osaurus:

- The approval rules. Decision 3 removes them.
- The plugin migration (`AppleAppsPluginMigration.swift`). We have no old
  plugins.

## 4. The tools and their operations

The tables give the op string, the verb that Multitool makes from it, and the
main arguments. The work item of each tool sets the full argument list and the
result type. Each argument name is camelCase.

### 4.1 `calendar` (EventKit)

| Op | Multitool verb | Main arguments |
|---|---|---|
| `list calendars` | `listCalendars` | — |
| `list events` | `listEvents` | `start`, `end`, `calendar?`, `query?`, `limit?` |
| `get event` | `getEvent` | `id`, `occurrenceStart?` |
| `create event` | `createEvent` | `calendar?`, `title`, `start`, `end`, `allDay?`, `location?`, `notes?`, `url?`, `alarms?`, `recurrence?` |
| `update event` | `updateEvent` | `id`, `occurrenceStart?`, `span?`, plus the fields of `create event` |
| `delete event` | `deleteEvent` | `id`, `occurrenceStart?`, `span?` |
| `open event` | `openEvent` | `id` |

### 4.2 `reminders` (EventKit)

| Op | Multitool verb | Main arguments |
|---|---|---|
| `list lists` | `listLists` | — |
| `list reminders` | `listReminders` | `list?`, `status?` (`incomplete`, `completed`, `all`), `dueBefore?`, `dueAfter?`, `query?`, `limit?` |
| `get reminder` | `getReminder` | `id` |
| `create reminder` | `createReminder` | `list?`, `title`, `notes?`, `url?`, `due?`, `priority?` |
| `update reminder` | `updateReminder` | `id`, plus the fields of `create reminder` |
| `complete reminder` | `completeReminder` | `id`, `completed?` (default `true`; `false` opens the reminder again) |
| `delete reminder` | `deleteReminder` | `id` |
| `open reminder` | `openReminder` | `id` |

### 4.3 `contacts` (Contacts.framework)

| Op | Multitool verb | Main arguments |
|---|---|---|
| `get me` | `getMe` | — |
| `search contacts` | `searchContacts` | `query`, `limit?` |
| `list contacts` | `listContacts` | `group?`, `limit?`, `offset?` |
| `list groups` | `listGroups` | — |
| `get contact` | `getContact` | `id` |
| `create contact` | `createContact` | `givenName?`, `familyName?`, `organization?`, `emails?`, `phones?`, `addresses?`, `note?`, `group?` |
| `update contact` | `updateContact` | `id`, plus the fields of `create contact` |
| `open contact` | `openContact` | `id` |

`list groups` is not in Osaurus. The live tests need it to find their own
group (see 12.2).

### 4.4 `notes` (AppleScript)

| Op | Multitool verb | Main arguments |
|---|---|---|
| `list folders` | `listFolders` | `account?` |
| `list notes` | `listNotes` | `folder?`, `limit?` |
| `search notes` | `searchNotes` | `query`, `folder?`, `limit?` |
| `get note` | `getNote` | `id` |
| `create note` | `createNote` | `folder?`, `title`, `body` |
| `append note` | `appendNote` | `id`, `body` |
| `open note` | `openNote` | `id` |

`get note` replaces `notes_read` of Osaurus, because `get` is the read verb of
all the other tools.

### 4.5 `mail` (AppleScript)

| Op | Multitool verb | Main arguments |
|---|---|---|
| `list mailboxes` | `listMailboxes` | `account?` |
| `list messages` | `listMessages` | `mailbox`, `unreadOnly?`, `limit?` |
| `get message` | `getMessage` | `id` |
| `search messages` | `searchMessages` | `query`, `mailbox?`, `field?` (`subject`, `sender`, `body`, `any`), `limit?` |
| `get thread` | `getThread` | `id` |
| `compose message` | `composeMessage` | `to`, `cc?`, `bcc?`, `subject`, `body`, `send?` (default `false`: open a draft) |
| `reply message` | `replyMessage` | `id`, `body`, `replyAll?`, `send?` (default `false`) |
| `move message` | `moveMessage` | `id`, `mailbox` |
| `mark message` | `markMessage` | `id`, `read?`, `flagged?` |

`mark message` replaces `mail_set_status` of Osaurus. `set status` gives the
verb `setStatus`, which does not tell the model which object changes.

### 4.6 `messages` (chat.db and AppleScript)

| Op | Multitool verb | Main arguments |
|---|---|---|
| `list conversations` | `listConversations` | `limit?` |
| `get conversation` | `getConversation` | `id`, `limit?`, `before?` |
| `list unread` | `listUnread` | `limit?` |
| `search messages` | `searchMessages` | `query`, `conversation?`, `limit?` |
| `send message` | `sendMessage` | `to` (handle or conversation id), `text` |

`get conversation` replaces `messages_read` of Osaurus.

### 4.7 `maps` (CoreLocation and MapKit)

| Op | Multitool verb | Main arguments |
|---|---|---|
| `get location` | `getLocation` | — |
| `geocode address` | `geocodeAddress` | `address` |
| `reverse_geocode coordinate` | `reverseGeocodeCoordinate` | `latitude`, `longitude` |
| `search places` | `searchPlaces` | `query`, `near?`, `limit?` |
| `explore places` | `explorePlaces` | `category`, `near?`, `limit?` |
| `get directions` | `getDirections` | `from?`, `to`, `transport?` (`driving`, `walking`, `transit`) |
| `get eta` | `getEta` | `from?`, `to`, `transport?` |
| `open map` | `openMap` | `query?`, `latitude?`, `longitude?`, `directionsTo?` |

The location operations of Osaurus (`location_*`) are in the `maps` tool,
because each app has one tool. When `from` or `near` is absent, the operation
uses the current location. `get location` waits a maximum of 120 seconds for
the permission dialog.

### 4.8 `music` (AppleScript)

| Op | Multitool verb | Main arguments |
|---|---|---|
| `get now_playing` | `getNowPlaying` | — |
| `control playback` | `controlPlayback` | `action` (`play`, `pause`, `toggle`, `next`, `previous`, `stop`) |
| `set volume` | `setVolume` | `level` (0 to 100) |
| `list playlists` | `listPlaylists` | — |
| `search library` | `searchLibrary` | `query`, `kind?` (`track`, `album`, `artist`, `playlist`), `limit?` |
| `play item` | `playItem` | `id` or `query`, `kind?`, `shuffle?` |

### 4.9 `shortcuts` (`/usr/bin/shortcuts`)

| Op | Multitool verb | Main arguments |
|---|---|---|
| `list shortcuts` | `listShortcuts` | `folder?` |
| `run shortcut` | `runShortcut` | `name`, `input?`, `timeoutSeconds?` |

### 4.10 Aliases

Each tool has a `verbAliases` table and a `nounAliases` table, as in
`CodeIndexTool`. No key of a table is a real verb or a real noun of that tool.
Examples:

- `calendar`: `find` → `list`, `add` → `create`, `remove` → `delete`,
  `edit` → `update`, `show` → `open`; `meeting` → `event`, `appointment` →
  `event`.
- `reminders`: `todo` → `reminder`, `task` → `reminder`, `finish` →
  `complete`, `check` → `complete`.
- `mail`: `send` → `compose`, `read` → `get`, `email` → `message`.
- `messages`: `text` → `send`, `chat` → `conversation`.
- `music`: `song` → `track` in `kind`, `playback` → `control`.

## 5. Shared rules

These rules are in `Support/` and all the tools use them.

1. **`ToolOutcome<T>` and `ToolSupport`.** We copy the shape of CodeContext:
   `.success(T)` or `.corrective(String)`, `parseChoice`, `choiceTable`,
   `outcome(of:)`. `AppleAppsError` replaces `CodeContextError`.
2. **`AppleAppsError`.** The cases are `notFound`, `invalidArgument`,
   `accessDenied(app:setting:)`, `appNotRunning`, `scriptFailed`,
   `timeout`, `unsupported`. `accessDenied` gives a correction that tells the
   model the name of the System Settings pane. The model can then tell the
   user what to enable.
3. **`DateArgument`.** It parses ISO 8601 with a time zone offset, ISO 8601
   with no offset (local time), and a bare date. It does not roll over a date.
   The time zone and the clock come from the context, thus the tests can
   inject them.
4. **`ClearableField<T>`.** It holds the three states of an update field:
   absent (no change), `null` (clear), and a value. A unit test proves that
   `@Generable` keeps the difference between absent and `null`. If it does
   not, see open question 1.
5. **Result types.** Each result is a flat `Encodable` struct. Each item has a
   stable `id`. Lists have `items` and `truncated`. A list operation has a
   default `limit` and a maximum `limit` in `AppleAppsDefaults`.
6. **`AppleScriptRunner`.** It runs a script through `/usr/bin/osascript` with
   `swift-subprocess`. It registers the child in `ProcessRegistry.global`. It
   gives each argument to the script as an `argv` item, not in the script
   text, thus no input can inject script code. It has a timeout, and it maps
   the error `-1743` (not authorized) to `accessDenied`.
7. **`ProcessRunner`.** It runs `/usr/bin/shortcuts` and `open` with
   `swift-subprocess`. It caps the output, and it cancels with SIGTERM, then
   SIGKILL.
8. **`open …` operations.** Calendar, Reminders, Contacts, Notes and Maps use
   their URL schemes (`ical://`, `x-apple-reminderkit://`, `addressbook://`,
   `maps://`), or AppleScript when no scheme exists. The opener is a
   protocol, thus a unit test records the URL and opens nothing.

## 6. Backends

Each app has one backend protocol, one live backend and one fake backend.

| App | Protocol | Live backend | Notes |
|---|---|---|---|
| Calendar | `CalendarBackend` | `EventKitCalendarBackend` | Shares one `EKEventStore` with Reminders. |
| Reminders | `RemindersBackend` | `EventKitRemindersBackend` | Same store as Calendar. |
| Contacts | `ContactsBackend` | `ContactStoreBackend` | Fetches only the keys that the result needs. |
| Notes | `NotesBackend` | `AppleScriptNotesBackend` | Note bodies are HTML. The backend converts to and from plain text. |
| Mail | `MailBackend` | `AppleScriptMailBackend` | Parses and escapes the mailbox path. |
| Messages | `MessagesBackend` | `ChatDatabaseMessagesBackend` | GRDB, read-only, on `~/Library/Messages/chat.db`. Sends through AppleScript. |
| Maps | `MapsBackend` | `MapKitMapsBackend` | `CLLocationManager`, `MKLocalSearch`, `MKDirections`, `CLGeocoder`. |
| Music | `MusicBackend` | `AppleScriptMusicBackend` | — |
| Shortcuts | `ShortcutsBackend` | `ShortcutsCLIBackend` | — |

`AppleAppsToolContext` holds the backend of each app, the opener, the clock
and the time zone. The public `make` functions make the live backends. An
`internal` initializer takes fakes for the unit tests.

### 6.1 Messages details

- `AttributedBodyDecoder` decodes the `attributedBody` column (an
  `NSArchiver` typedstream) when the `text` column is empty. Search uses the
  decoded text.
- `send message` sends through AppleScript with iMessage. It then polls
  `chat.db` for the new row and reads `is_delivered` and `error`. When
  iMessage fails, it sends again through SMS. When the result is not known
  after the timeout, it does not send again. The result then tells that the
  state is not known.
- The process needs Full Disk Access to open `chat.db`. When the open fails,
  the operation gives the `accessDenied` correction.

## 7. Public API

```swift
/// The nine Apple apps that this package has a tool for.
public enum AppleApp: String, CaseIterable, Sendable, Codable {
    case calendar, reminders, contacts, notes, mail, messages, maps, music, shortcuts
}

/// The FoundationModels tools of this package.
public enum AppleAppsTools {
    /// The name of the tool of each app. The name is the raw value of the app.
    public static let toolNames: [AppleApp: String]

    /// The op strings of each tool, in the order of its fused schema.
    public static let operationNames: [AppleApp: [String]]

    /// Makes the tool of one app.
    public static func make(
        app: AppleApp,
        configuration: AppleAppsConfiguration = .init(),
        includesSchemaInInstructions: Bool = AppleAppsDefaults.includesSchemaInInstructions
    ) throws -> any Tool

    /// Makes the tools of the given apps, in the order of `AppleApp.allCases`.
    public static func make(
        apps: Set<AppleApp>,
        configuration: AppleAppsConfiguration = .init(),
        includesSchemaInInstructions: Bool = AppleAppsDefaults.includesSchemaInInstructions
    ) throws -> [any Tool]
}

/// Settings that apply to all the tools.
public struct AppleAppsConfiguration: Sendable {
    public var timeZone: TimeZone        // default: .current
    public var scriptTimeout: Duration   // default: 30 seconds
    public var shortcutsOutputLimit: Int // default: 1 MiB
}
```

## 8. Package layout

```
FoundationModelsAppleApps/
  Package.swift
  README.md
  plan.md
  docs/
    SECURITY.md                  # what each tool can read and change
  Sources/FoundationModelsAppleApps/
    AppleApp.swift
    AppleAppsTools.swift         # the public entry point
    AppleAppsConfiguration.swift
    AppleAppsDefaults.swift
    AppleAppsToolContext.swift
    Support/
      ToolSupport.swift          # ToolOutcome, parseChoice, outcome(of:)
      AppleAppsError.swift
      DateArgument.swift
      ClearableField.swift
      AppleScriptRunner.swift
      ProcessRunner.swift
      URLOpener.swift
    Calendar/
      CalendarTool.swift         # name, description, aliases, operations(), make
      CalendarBackend.swift      # protocol and result types
      EventKitCalendarBackend.swift
      ListCalendarsOperation.swift
      ListEventsOperation.swift
      ...                        # one file for each operation
    Reminders/   ...             # same shape
    Contacts/    ...
    Notes/       ...             # plus NotesScripts.swift (AppleScript text)
    Mail/        ...             # plus MailScripts.swift, MailboxPath.swift
    Messages/    ...             # plus ChatDatabase.swift, AttributedBodyDecoder.swift
    Maps/        ...
    Music/       ...             # plus MusicScripts.swift
    Shortcuts/   ...
  Tests/FoundationModelsAppleAppsTests/
    Fakes/                       # one fake backend for each app, FakeOpener, FakeScriptRunner
    Fixtures/chat.db             # a small synthetic Messages database
    Goldens/                     # op names and fused schema for each tool
    <App>/<Op>Tests.swift
  IntegrationTests/
    Package.swift                # .package(path: ".."), plus Multitool
    Tests/FoundationModelsAppleAppsIntegrationTests/
      Live/<App>LiveTests.swift  # level 2
      Scenarios/<App>ScenarioTests.swift  # level 3
  Examples/AppleAppsExample/     # a CLI that mounts the tools and runs one prompt
```

## 9. Package.swift

- `// swift-tools-version: 6.2` and `platforms: [.macOS("27.0")]`.
- One library product and one target: `FoundationModelsAppleApps`.
- Each dependency name is a `private let` constant with a doc comment that
  tells why the package needs it. Family packages use the helper
  `swissArmyHammerPackage(name:)` with a URL and `branch: "main"`.
- Dependencies:
  - FoundationModelsExtras: products `Operations` and `FoundationModelsExtras`.
  - `swift-subprocess`, `exact: "1.0.0-beta.1"`, the same pin as Multitool.
  - GRDB, `from: "7.0.0"`, the same as CodeContext.
  - `swift-log` for telemetry. The library does not link `swift-otel`.
- `PackageManifestTests` checks that the library does not link Multitool and
  does not link `swift-otel`.
- The examples are `.executableTarget`s under `Examples/`.
- `IntegrationTests/` is a separate package. It depends on the root with
  `.package(path: "..")` and on FoundationModelsMultitool.

## 10. Mount in Multitool and in the agent

### 10.1 Multitool

No change to Multitool is necessary. A host mounts each tool as its own group:

```swift
let builder = MultiTool.Builder()
for app in enabledApps {
    builder.addGroup(named: app.rawValue, [try AppleAppsTools.make(app: app)])
}
```

`RegistrySource` expands each `OperationTool` into verbs:
`tools.calendar.listEvents`, `tools.mail.composeMessage`. A test in the
integration package proves that no group name conflicts with a noun of
Multitool (`files`, `web`, `git`, `environment`, `shell`, `code_context`).

### 10.2 FoundationModelsACPAgent

1. **Package.swift.** Add the URL dependency on FoundationModelsAppleApps.
2. **`Configuration/ToolSectionCodec.swift`.** Add the section
   `appleApps: ToolSection<AppleAppsToolOptions>`. Add it to `optionKeys` and
   to `init(from:)`. `AppleAppsToolOptions` has one `Bool` for each app. All
   are `false` by default. Thus an absent section or `appleApps: {}` mounts no
   app, and the user turns on each app by name:

   ```yaml
   appleApps:
     calendar: true
     reminders: true
     mail: true
   ```

   An unknown key is an error, through `KeyCheckedSection`.
3. **`Tools/ToolCatalog.swift`.** Add `composeAppleApps`. It calls
   `builder.addGroup(named:_:)` once for each enabled app.
4. **Info.plist.** The agent is a CLI. EventKit, Contacts and CoreLocation
   need usage strings in the `Info.plist` of the process. The agent embeds an
   `Info.plist` in the binary with the linker flag
   `-sectcreate __TEXT __info_plist`. The keys are
   `NSCalendarsFullAccessUsageDescription`,
   `NSRemindersFullAccessUsageDescription`, `NSContactsUsageDescription`,
   `NSLocationWhenInUseUsageDescription` and `NSAppleEventsUsageDescription`.
   The example of this package does the same.
5. **README.** Add one row for each app to the table in § Tools.

## 11. Permissions (TCC)

| Tool | macOS permission | When the permission is missing |
|---|---|---|
| `calendar` | Calendars, Full Access | `requestFullAccessToEvents()` on first use, then `accessDenied` |
| `reminders` | Reminders, Full Access | `requestFullAccessToReminders()` on first use, then `accessDenied` |
| `contacts` | Contacts | `requestAccess(for: .contacts)` on first use, then `accessDenied` |
| `notes` | Automation → Notes | macOS asks on the first script. Error `-1743` gives `accessDenied`. |
| `mail` | Automation → Mail | Same as `notes` |
| `messages` | Full Disk Access, and Automation → Messages | The open of `chat.db` fails, which gives `accessDenied`. macOS cannot ask for Full Disk Access. |
| `maps` | Location | `requestWhenInUseAuthorization()`, wait a maximum of 120 seconds |
| `music` | Automation → Music | Same as `notes` |
| `shortcuts` | None. A shortcut can ask for its own permissions. | — |

macOS gives the permission to the responsible process. For a CLI agent that
is the terminal app or the app that started the agent. `docs/SECURITY.md` and
the README tell this.

## 12. Testing

All tests use Swift Testing (`@Suite`, `@Test`, `#expect`). The root
`swift test` runs only level 1. No test uses an environment variable to
select tests.

### 12.1 Level 1: unit tests, no TCC, no model

| Suite | What it proves |
|---|---|
| `<App>ToolTests` | The tool fuses its schema. The op names agree with the golden. No alias key is a real verb or noun. |
| `<App><Op>Tests` | Each operation with a fake backend: the arguments go to the backend, the result encodes as expected, and each recoverable error gives a correction. |
| `DateArgumentTests` | ISO 8601 forms, bare dates, no roll-over, inclusive all-day end, injected time zone. |
| `ClearableFieldTests` | Absent, `null` and a value stay different through `@Generable`. |
| `AppleScriptRunnerTests` | Arguments go as `argv`, never in the script text. `-1743` gives `accessDenied`. The timeout stops the child. Uses a fake `osascript` script in a temporary directory. |
| `MailboxPathTests` | `Account/Folder/Subfolder` and `\/`. |
| `ChatDatabaseTests` | Reads of the synthetic `Fixtures/chat.db`, including `attributedBody`. |
| `SendMessageTests` | Delivered, failed then SMS, and not known (no second send), with a fake database and a fake sender. |
| `ShortcutsCLIBackendTests` | Output cap and `truncated`, SIGTERM then SIGKILL. Uses a fake CLI script. |
| `PackageManifestTests` | No Multitool and no `swift-otel` in the library. |

### 12.2 Level 2: live, no model

These tests are in the `IntegrationTests` package. Each suite has the trait
`.enabled(if: AppleAppsAccess.isGranted(.<app>), "…")`. Thus a machine with no
permission skips the suite and does not fail it. Each suite also has a
`.timeLimit` as a hang guard.

The tests change only their own containers. Each suite makes the container
when it is absent, and removes each item that it made:

| App | Container | Live writes |
|---|---|---|
| Calendar | Calendar `FMAppleApps Test` | Create, update, delete events, including a recurring event |
| Reminders | List `FMAppleApps Test` | Create, update, complete, delete |
| Contacts | Group `FMAppleApps Test` | Create and update contacts. The suite removes them at the end. |
| Notes | Folder `FMAppleApps Test` | Create and append |
| Mail | — | Read only. `compose message` with `send: false` only, then the draft is closed. |
| Messages | — | Read only. No `send message`. |
| Maps | — | Read only (geocode, search, directions) |
| Music | — | Read only (`get now_playing`, `list playlists`, `search library`) |
| Shortcuts | — | `list shortcuts` only, unless a shortcut named `FMAppleApps Test` exists, then `run shortcut` with it |

### 12.3 Level 3: one real-model scenario for each app

Each scenario mounts one tool in `MultiTool.Builder` with `addGroup`, and
runs one short prompt through `runGatedTurnScenario`. The checks are code
properties only: the expected verb was called, no correction came back, and
the answer has an injected value. The scenarios use the same containers as
level 2. An offline suite in the same file tests the checks.

## 13. Work items

Each work item is one kanban task with the tag `apple-apps`. Each item ends
with a green `swift test` and a green
`swift build --package-path IntegrationTests --build-tests`.

1. **Package skeleton.** `Package.swift`, the empty target, the test target,
   the `IntegrationTests` package, `PackageManifestTests`, a minimum README.
2. **Shared support.** `ToolSupport`, `ToolOutcome`, `AppleAppsError`,
   `DateArgument`, `ClearableField`, `AppleAppsDefaults`,
   `AppleAppsConfiguration`, `URLOpener`. Includes the `@Generable` null test.
3. **Runners.** `AppleScriptRunner` and `ProcessRunner`, with their tests.
4. **Public entry point.** `AppleApp`, `AppleAppsToolContext`,
   `AppleAppsTools` with no apps yet, and `AppleAppsAccess`.
5. **Calendar.** Backend, operations, aliases, tool, unit tests, live tests.
6. **Reminders.** Same shape as Calendar. Shares the `EKEventStore`.
7. **Contacts.**
8. **Notes.**
9. **Mail.**
10. **Messages, read.** `ChatDatabase`, `AttributedBodyDecoder`, the
    synthetic fixture, the four read operations.
11. **Messages, send.** `send message` with the delivery check.
12. **Maps.**
13. **Music.**
14. **Shortcuts.**
15. **Multitool proof.** An integration test that mounts all nine tools in
    `MultiTool.Builder`, checks the rendered TypeScript surface against a
    golden, and runs one `runCode` script that calls one verb of each tool
    with fake backends.
16. **Scenarios.** One level 3 scenario for each app.
17. **Example.** `Examples/AppleAppsExample`, with the embedded `Info.plist`.
18. **Docs.** The full README (one section for each app, with its permission)
    and `docs/SECURITY.md`.
19. **Agent wiring.** The changes of 10.2 in FoundationModelsACPAgent.

Items 5 to 14 depend only on items 1 to 4. Thus they can run in parallel.

## 14. Decisions

1. **Operation tools, one tool for each app. — DECIDED (2026-10-14).** Each
   app is one `OperationTool` with its own verbs and nouns, as in
   CodeContext. Multitool is the expected host.
2. **All nine apps in one phase. — DECIDED (2026-10-14).** The work items are
   one flat list.
3. **No permission gate. — DECIDED (2026-10-14).** No operation asks for
   approval, and there is no `readOnly` flag. The host selects the apps.
   Send and delete operations run as all other operations.
4. **The agent wiring is in this plan, with one key for each app. —
   DECIDED (2026-10-14).** All apps are off by default.
5. **Live tests write only to their own containers. — DECIDED (2026-10-14).**
   Mail and Messages live tests only read and never send.
6. **One library. — DECIDED (2026-10-14).** One target and one product, with
   one folder for each app.
7. **AppleScript through `osascript`. — DECIDED (2026-10-14).** Not
   `NSAppleScript`. `NSAppleScript` must run on one thread and cannot be
   cancelled. A child process can have a timeout and can be stopped, and
   `swift-subprocess` is already in the family.
8. **GRDB for `chat.db`. — DECIDED (2026-10-14).** The same library as
   CodeContext. The database opens read-only.
9. **MapKit for Maps. — DECIDED (2026-10-14).** Osaurus does not name its
   maps backend. MapKit needs no API key.

## 15. Open questions

1. **Clear a field with `null`.** If `@Generable` does not keep the
   difference between absent and `null`, the update operations get a `clear`
   argument (a list of field names) in place of `null`. Work item 2 answers
   this.
2. **IDs from AppleScript.** Notes and Mail give IDs that can change when an
   item moves. Work items 8 and 9 measure this. If an ID changes, the result
   of `move message` gives the new ID.

## 16. Out of scope

- Apps that are not in the list: Photos, Safari, Finder, Home, Files.
- iOS and iPadOS. The package is for macOS only.
- An approval or policy layer (decision 3).
- Mail attachments and Messages attachments, other than their names.

## 17. References

- Osaurus Apple apps: <https://github.com/osaurus-ai/osaurus/blob/main/docs/APPLE_APPS.md>
- Osaurus source: `Packages/OsaurusCore/AppleApps/` (`AppleApp.swift`,
  `AppleAppToolCatalog.swift`, one folder for each app).
- The operation pattern: `FoundationModelsCodeContext/Sources/FoundationModelsCodeContext/Tools/`.
- The `Operations` library: `FoundationModelsExtras/Sources/Operations/`.
- The mount of operation tools: `FoundationModelsMultitool/Sources/FoundationModelsMultitool/Surface/RegistrySource.swift`
  and `OperationVerbTool.swift`.
- The plan format: `FoundationModelsMultitool/web.md`, `FoundationModelsKanban/plan.md`.
- The agent mount: `FoundationModelsACPAgent/Sources/FoundationModelsACPAgent/Tools/ToolCatalog.swift`
  and `Configuration/ToolSectionCodec.swift`.
