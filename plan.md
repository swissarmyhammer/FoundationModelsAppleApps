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
let tools = try AppleAppsTools.make(apps: [.calendar, .reminders, .mail], mode: .open)
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
9. **Two modes: `open` and `sandboxed`.** The same as MacControl (D128 of the
   TestudoApp design). OpenTestudo and the ACP agent run in `open` mode, with
   all operations. Testudo is on the Mac App Store, in the App Sandbox, and
   runs in `sandboxed` mode. In that mode a tool registers only the
   operations that work in the sandbox (the "Sandbox" column of each table in
   section 4, and section 4.11). The model never sees an operation that
   cannot work. A tool with no such operation is not registered.

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
| Shortcuts | `/usr/bin/shortcuts` | `/usr/bin/shortcuts` through `ProcessRunner` of FoundationModelsExtras |

We copy these rules from Osaurus:

- Dates are ISO 8601. A bare date is valid. An invalid date gives a
  correction. A date does not roll over (`2026-02-30` is not valid).
- The `end` of an all-day event is inclusive.
- An absent field does not change on update. To clear a field, the update
  operation names it in `clear` (decision 10). Osaurus uses JSON `null` for
  this. We cannot (see 5.4).
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
  `truncated: true`.

We do not copy these from Osaurus:

- The approval rules. Decision 3 removes them.
- SIGTERM before SIGKILL on a cancel. We use `ProcessRunner` of Extras, which
  kills the process group with SIGKILL on a timeout or a cancel (see 5.7).
- The plugin migration (`AppleAppsPluginMigration.swift`). We have no old
  plugins.

## 4. The tools and their operations

The tables give the op string, the verb that Multitool makes from it, the
main arguments, and whether the operation is registered in `sandboxed` mode
(principle 9, section 4.11). The work item of each tool sets the full
argument list and the result type. Each argument name is camelCase.

### 4.1 `calendar` (EventKit)

| Op | Multitool verb | Main arguments | Sandbox |
|---|---|---|---|
| `list calendars` | `listCalendars` | — | Yes |
| `list events` | `listEvents` | `start`, `end`, `calendar?`, `query?`, `limit?` | Yes |
| `get event` | `getEvent` | `id`, `occurrenceStart?` | Yes |
| `create event` | `createEvent` | `calendar?`, `title`, `start`, `end`, `allDay?`, `location?`, `notes?`, `url?`, `alarms?`, `recurrence?` | Yes |
| `update event` | `updateEvent` | `id`, `occurrenceStart?`, `span?`, `clear?` (`location`, `notes`, `url`, `alarms`, `recurrence`), plus the fields of `create event` | Yes |
| `delete event` | `deleteEvent` | `id`, `occurrenceStart?`, `span?` | Yes |
| `open event` | `openEvent` | `id` | Yes (URL scheme) |

### 4.2 `reminders` (EventKit)

| Op | Multitool verb | Main arguments | Sandbox |
|---|---|---|---|
| `list lists` | `listLists` | — | Yes |
| `list reminders` | `listReminders` | `list?`, `status?` (`incomplete`, `completed`, `all`), `dueBefore?`, `dueAfter?`, `query?`, `limit?` | Yes |
| `get reminder` | `getReminder` | `id` | Yes |
| `create reminder` | `createReminder` | `list?`, `title`, `notes?`, `url?`, `due?`, `priority?` | Yes |
| `update reminder` | `updateReminder` | `id`, `clear?` (`notes`, `url`, `due`, `priority`), plus the fields of `create reminder` | Yes |
| `complete reminder` | `completeReminder` | `id`, `completed?` (default `true`; `false` opens the reminder again) | Yes |
| `delete reminder` | `deleteReminder` | `id` | Yes |
| `open reminder` | `openReminder` | `id` | Yes (URL scheme) |

### 4.3 `contacts` (Contacts.framework)

| Op | Multitool verb | Main arguments | Sandbox |
|---|---|---|---|
| `get me` | `getMe` | — | Yes |
| `search contacts` | `searchContacts` | `query`, `limit?` | Yes |
| `list contacts` | `listContacts` | `group?`, `limit?`, `offset?` | Yes |
| `list groups` | `listGroups` | — | Yes |
| `get contact` | `getContact` | `id` | Yes |
| `create contact` | `createContact` | `givenName?`, `familyName?`, `organization?`, `emails?`, `phones?`, `addresses?`, `note?`, `group?` | Yes |
| `update contact` | `updateContact` | `id`, `clear?` (`organization`, `emails`, `phones`, `addresses`, `note`), plus the fields of `create contact` | Yes |
| `open contact` | `openContact` | `id` | Yes (URL scheme) |

`list groups` is not in Osaurus. The live tests need it to find their own
group (see 12.2).

### 4.4 `notes` (AppleScript)

| Op | Multitool verb | Main arguments | Sandbox |
|---|---|---|---|
| `list folders` | `listFolders` | `account?` | No |
| `list notes` | `listNotes` | `folder?`, `limit?` | No |
| `search notes` | `searchNotes` | `query`, `folder?`, `limit?` | No |
| `get note` | `getNote` | `id` | No |
| `create note` | `createNote` | `folder?`, `title`, `body` | No |
| `append note` | `appendNote` | `id`, `body` | No |
| `open note` | `openNote` | `id` | No (see below) |

`get note` replaces `notes_read` of Osaurus, because `get` is the read verb of
all the other tools.

Notes publishes only the scripting access group `com.apple.Notes.openlocation`.
`open note` could use it, but in `sandboxed` mode no other operation gives a
note `id`. Thus the `notes` tool is not registered in `sandboxed` mode.

### 4.5 `mail` (AppleScript)

| Op | Multitool verb | Main arguments | Sandbox |
|---|---|---|---|
| `list mailboxes` | `listMailboxes` | `account?` | No |
| `list messages` | `listMessages` | `mailbox`, `unreadOnly?`, `limit?` | No |
| `get message` | `getMessage` | `id` | No |
| `search messages` | `searchMessages` | `query`, `mailbox?`, `field?` (`subject`, `sender`, `body`, `any`), `limit?` | No |
| `get thread` | `getThread` | `id` | No |
| `compose message` | `composeMessage` | `to`, `cc?`, `bcc?`, `subject`, `body`, `send?` (default `false`: open a draft) | Yes (`com.apple.mail.compose`) |
| `reply message` | `replyMessage` | `id`, `body`, `replyAll?`, `send?` (default `false`) | No |
| `move message` | `moveMessage` | `id`, `mailbox` | No |
| `mark message` | `markMessage` | `id`, `read?`, `flagged?` | No |

`mark message` replaces `mail_set_status` of Osaurus. `set status` gives the
verb `setStatus`, which does not tell the model which object changes.

Mail publishes only the scripting access group `com.apple.mail.compose`, for
the outgoing message. Thus only `compose message` works in `sandboxed` mode.
Whether the `send` command is in that group is **unverified** (work item 21).
When it is not, `compose message` in `sandboxed` mode has no `send` argument
and always opens a draft.

### 4.6 `messages` (chat.db and AppleScript)

| Op | Multitool verb | Main arguments | Sandbox |
|---|---|---|---|
| `list conversations` | `listConversations` | `limit?` | No |
| `get conversation` | `getConversation` | `id`, `limit?`, `before?` | No |
| `list unread` | `listUnread` | `limit?` | No |
| `search messages` | `searchMessages` | `query`, `conversation?`, `limit?` | No |
| `send message` | `sendMessage` | `to` (handle or conversation id), `text` | No |

`get conversation` replaces `messages_read` of Osaurus.

Messages publishes no scripting access group, so a sandboxed app cannot send.
The sandbox blocks reads of `~/Library/Messages`, and Full Disk Access does not
remove that block. Thus the `messages` tool is not registered in `sandboxed`
mode (open question 2).

### 4.7 `maps` (CoreLocation and MapKit)

| Op | Multitool verb | Main arguments | Sandbox |
|---|---|---|---|
| `get location` | `getLocation` | — | Yes |
| `geocode address` | `geocodeAddress` | `address` | Yes |
| `reverse_geocode coordinate` | `reverseGeocodeCoordinate` | `latitude`, `longitude` | Yes |
| `search places` | `searchPlaces` | `query`, `near?`, `limit?` | Yes |
| `explore places` | `explorePlaces` | `category`, `near?`, `limit?` | Yes |
| `get directions` | `getDirections` | `from?`, `to`, `transport?` (`driving`, `walking`, `transit`) | Yes |
| `get eta` | `getEta` | `from?`, `to`, `transport?` | Yes |
| `open map` | `openMap` | `query?`, `latitude?`, `longitude?`, `directionsTo?` | Yes (URL scheme) |

The location operations of Osaurus (`location_*`) are in the `maps` tool,
because each app has one tool. When `from` or `near` is absent, the operation
uses the current location. `get location` waits a maximum of 120 seconds for
the permission dialog.

### 4.8 `music` (AppleScript)

| Op | Multitool verb | Main arguments | Sandbox |
|---|---|---|---|
| `get now_playing` | `getNowPlaying` | — | Yes (`com.apple.Music.playback`) |
| `control playback` | `controlPlayback` | `action` (`play`, `pause`, `toggle`, `next`, `previous`, `stop`) | Yes (`com.apple.Music.playback`) |
| `set volume` | `setVolume` | `level` (0 to 100) | Yes (`com.apple.Music.playback`, **unverified**) |
| `list playlists` | `listPlaylists` | — | Yes (`com.apple.Music.library.read`) |
| `search library` | `searchLibrary` | `query`, `kind?` (`track`, `album`, `artist`, `playlist`), `limit?` | Yes (`com.apple.Music.library.read`) |
| `play item` | `playItem` | `id` or `query`, `kind?`, `shuffle?` | Yes (`com.apple.Music.playback`) |

### 4.9 `shortcuts` (`/usr/bin/shortcuts`; Apple events in `sandboxed` mode)

| Op | Multitool verb | Main arguments | Sandbox |
|---|---|---|---|
| `list shortcuts` | `listShortcuts` | `folder?` | Yes (`com.apple.shortcuts.run`) |
| `run shortcut` | `runShortcut` | `name`, `input?`, `timeoutSeconds?` | Yes (`com.apple.shortcuts.run`) |

In `sandboxed` mode the tool cannot start `/usr/bin/shortcuts`, because a
child process inherits the sandbox. The sandboxed backend sends Apple events
to **Shortcuts Events** (`com.apple.shortcuts.events`) instead. This is the
path that Apple gives for apps:

- **Apple says to use it.** WWDC21 session 10232 ("Meet Shortcuts for
  macOS") says that an app that runs shortcuts must use the scripting
  interface of the Shortcuts Events process, not the command-line tool. A
  sandboxed app adds `com.apple.security.scripting-targets` with the
  `com.apple.shortcuts.run` access group. The session shows the Scripting
  Bridge code: `SBApplication(bundleIdentifier: "com.apple.shortcuts.events")`,
  the `shortcuts` element, and `run(withInput:) -> Any?`.
- **The target key is `com.apple.shortcuts.events`.** An Apple engineer on the
  Developer Forums (thread 698879) says that the key in the
  `scripting-targets` dictionary must be the bundle id of Shortcuts Events. The
  forum reports that the WWDC slide had a wrong key, and that App Store upload
  refused that wrong key.
- **The `sdef` agrees (macOS 27, 2026-10-23).** Shortcuts Events uses the
  same `Shortcuts.sdef` as the Shortcuts app. The `shortcut` and `folder`
  elements, and their properties (`name`, `id`, `folder`, `accepts input`),
  are readable in `com.apple.shortcuts.run`. The `run` command is in
  `com.apple.shortcuts.run`, takes `with input` (any type) and returns the
  result of the shortcut. Thus `list shortcuts` and its `folder` filter work
  in `sandboxed` mode. `com.apple.shortcuts.organize` (change a folder) is not
  needed.
- **Shortcuts Events runs the shortcut in the background.** The Shortcuts
  app does not open.
- **The user approves one Automation prompt** ("… wants to control Shortcuts
  Events"), with the text of `NSAppleEventsUsageDescription`. A denied prompt
  gives the `-1743` error; the tool returns a correction that tells the user to
  allow it in System Settings > Privacy & Security > Automation.
- **A shortcut runs outside the sandbox of the host.** It runs in the process
  of Shortcuts Events, with the permissions of Shortcuts. Thus a shortcut that
  the user made can do work that the host cannot do. The risk check of the host
  must treat `run shortcut` as an operation whose effect it cannot know before
  the run. The package does not install shortcuts (decision 3).
- **Not used:** the `shortcuts://x-callback-url/run-shortcut` URL scheme. It
  works in the sandbox, but it opens the Shortcuts app and returns the result
  only through a callback URL.

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

### 4.11 The `sandboxed` mode

A sandboxed app (the App Sandbox of the Mac App Store) has these limits:

- **Child processes inherit the sandbox.** A child does not get the
  entitlements of the app. Thus `/usr/bin/osascript` and `/usr/bin/shortcuts`
  cannot send Apple events or read outside the container for the app.
- **Apple events need a scripting access group.** The app sends Apple events
  in its own process, with the entitlement
  `com.apple.security.scripting-targets`. Each target app lists the groups
  that it publishes in its `sdef`. The other path, the entitlement
  `com.apple.security.temporary-exception.apple-events`, is not used: App
  Review usually rejects it.
- **Frameworks with TCC work.** EventKit, Contacts and CoreLocation work with
  their `personal-information` entitlements. MapKit needs
  `com.apple.security.network.client`.
- **URL schemes work.** `NSWorkspace.open(_:)` needs no entitlement.

The scripting access groups on macOS 27, from `sdef` on each app (2026-10-23):

| App | Access groups |
|---|---|
| Mail | `com.apple.mail.compose` |
| Notes | `com.apple.Notes.openlocation` |
| Music | `com.apple.Music.library.read`, `com.apple.Music.library.read-write`, `com.apple.Music.playback`, `com.apple.Music.user-interface` |
| Shortcuts | `com.apple.shortcuts.run`, `com.apple.shortcuts.organize` |
| Calendar | `com.apple.iCal.read`, `com.apple.iCal.read-write`, `com.apple.iCal.UI` (not used: EventKit is better) |
| Messages | none |

The result in `sandboxed` mode:

| Tool | Registered | Operations |
|---|---|---|
| `calendar`, `reminders`, `contacts`, `maps` | Yes | All |
| `music` | Yes | All, through in-process Apple events (5.9) |
| `shortcuts` | Yes | All, through in-process Apple events (5.9) |
| `mail` | Yes | `compose message` only |
| `notes` | No | — |
| `messages` | No | — |

Each tool has the static list of its sandboxed operations. In `sandboxed` mode,
`make` builds the `OperationTool` with only those operations. Thus the fused
schema, the op names and the Multitool verbs show only what works. A request
for a tool with no sandboxed operation (`notes`, `messages`) gives no tool and
no error: `make(apps:)` leaves it out, and `make(app:)` returns `nil`.

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
4. **`clear` on update.** Each update operation has the argument
   `clear: [String]?`. It is a list of the names of the fields to clear, for
   example `updateEvent({ id, clear: ["location", "notes"] })`. A shared
   helper, `ClearList`, parses the names with `parseChoice` against the
   clearable fields of the operation. An unknown name gives a correction that
   lists the allowed names. When a call gives a value for a field and also
   names that field in `clear`, the call gives a correction and changes
   nothing.

   JSON `null` cannot clear a field, for these reasons from the existing code:
   - Multitool keeps a JS `null` different from an absent key
     (`Invocation/ArgumentMarshaler.swift`), and `OperationResolver.resolveParameters`
     in Extras keeps a present `null` key.
   - But `AnyOperation` then decodes the payload into the `@Generable` struct.
     That decode changes a `null` into `nil`, the same as an absent key
     (`Invocation/ToolInvoker.swift`, the documentation of the `null` check).
   - No generated schema declares a nullable type. Thus a native tool call,
     with guided generation, cannot give `null`.
5. **Result types.** Each result is a flat `Encodable` struct. Each item has a
   stable `id`. Lists have `items` and `truncated`. A list operation has a
   default `limit` and a maximum `limit` in `AppleAppsDefaults`.
6. **`AppleScriptRunner`.** It runs a script through `/usr/bin/osascript` with
   `ProcessRunner` of Extras (see 5.7). It gives each argument to the script
   as an `argv` item, not in the script text, thus no input can inject script
   code. It has a timeout, and it maps the error `-1743` (not authorized) to
   `accessDenied`.
7. **Child processes.** The package has no process runner of its own. It uses
   the public `ProcessRunner` of the `FoundationModelsExtras` product for
   `osascript`, `/usr/bin/shortcuts` and `open`. That runner spawns the child
   in its own process group and keeps the pid in `ProcessRegistry.global`
   while the child runs. It caps the output (`OutputCap`) and tells when it
   cut the output (`isTruncated`). On a timeout or a cancel, it kills the
   group with SIGKILL and gives `.timedOut`. It merges stdout and stderr, thus
   the exit code tells a failure, and the output is then the error text.
8. **`open …` operations.** Calendar, Reminders, Contacts, Notes and Maps use
   their URL schemes (`ical://`, `x-apple-reminderkit://`, `addressbook://`,
   `maps://`), or AppleScript when no scheme exists. The opener is a
   protocol, thus a unit test records the URL and opens nothing.
9. **`AppleEventRunner`, for `sandboxed` mode.** One protocol with two live
   implementations. `OsascriptRunner` is the `AppleScriptRunner` of 5.6, for
   `open` mode. `InProcessAppleEventRunner` is for `sandboxed` mode. It runs a
   compiled `NSAppleScript` with a handler call
   (`executeAppleEvent(_:error:)`), and gives the arguments as an
   `NSAppleEventDescriptor` list, never in the script text. `NSAppleScript`
   is not thread-safe. Thus the runner runs all scripts on one dedicated
   thread, one at a time. It cannot cancel a script that runs. Thus it
   sets the Apple event timeout (`with timeout of N seconds` in each handler)
   to `scriptTimeout`, and gives `timeout` when it ends. It maps `-1743` to
   `accessDenied` and `-10004` (privilege violation: the event is not in a
   granted access group) to `unsupported`, with a correction that names the
   sandbox. The Music and Shortcuts backends take the runner, so the same
   backend works in both modes. The Shortcuts backend has two
   implementations: `ShortcutsCLIBackend` (`open`) and
   `ShortcutsAppleEventBackend` (`sandboxed`).

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
| Music | `MusicBackend` | `AppleScriptMusicBackend` | Takes an `AppleEventRunner` (5.9), so it works in both modes. |
| Shortcuts | `ShortcutsBackend` | `ShortcutsCLIBackend` (`open`), `ShortcutsAppleEventBackend` (`sandboxed`) | — |

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

### 6.2 IDs from AppleScript

We copy the design of Osaurus (`MailService.swift` and `NotesService.swift`
in `Packages/OsaurusCore/AppleApps/`).

**Mail.**

- The `id` of a message is its RFC Message-ID. It is globally unique, and it
  does not change when the message moves.
- Mail's internal id is not an `id` for the model. In AppleScript,
  `message id` is the RFC Message-ID property, thus the script finds the
  internal id with `first message of <mailbox> whose id is <n>`.
- `MailMessageCache` maps a Message-ID to the mailbox path and the internal
  id. Thus a later call does not scan the mailbox again. It holds a maximum of
  512 entries. When it is full, it removes the oldest entry. Osaurus does not
  move an entry on a hit, thus its cache is first-in, first-out. Our cache
  also moves an entry to the end on a hit (LRU). This is the only change from
  Osaurus.
- To find a message, the backend tries these steps in this order:
  1. The cached mailbox and internal id. The backend then checks that the
     Message-ID of the result is the same. If it is not, the hit is stale.
  2. A scan of the given mailbox, then of the cached mailbox, by Message-ID.
  3. A scan of the inbox by Message-ID.
- When no step finds the message (error `-1728`), the backend removes the
  cache entry and gives `notFound`.
- `move message` moves the message, then polls the destination by
  Message-ID (10 times, 0.3 seconds apart). It then stores the new mailbox
  and internal id in the cache. The result keeps the same `id`. When the poll
  times out, the result is the row from before the move, with the
  destination as its mailbox. A move to the unified Trash (`Trash`, with no
  account) uses `delete`.

**Notes.**

- The `id` of a note or a folder is its Core Data id
  (`x-coredata://…/ICNote/pNNN`). The backend finds a note with
  `note id <id>`.
- After `make new note`, Notes gives a provisional id. When Notes saves the
  note, the same object gives its permanent id. Thus `create note` polls the
  id (6 times, 0.25 seconds apart) until it changes, and returns the
  permanent id.
- An `id` that finds no note gives the `notFound` correction. The correction
  tells the model to call `list notes` or `search notes` and use one of their
  `id` values.
- A `folder` argument that starts with `x-coredata://` is a folder id. Any
  other `folder` value is a folder name.

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

    /// The op strings of each tool in the given mode, in the order of its
    /// fused schema. An app with no operation in the mode is absent.
    public static func operationNames(in mode: HostMode) -> [AppleApp: [String]]

    /// Makes the tool of one app, or `nil` when the app has no operation in
    /// the mode (in `sandboxed` mode: `notes`, `messages`).
    public static func make(
        app: AppleApp,
        mode: HostMode,
        configuration: AppleAppsConfiguration = .init(),
        includesSchemaInInstructions: Bool = AppleAppsDefaults.includesSchemaInInstructions
    ) throws -> (any Tool)?

    /// Makes the tools of the given apps, in the order of `AppleApp.allCases`.
    /// An app with no operation in the mode is left out.
    public static func make(
        apps: Set<AppleApp>,
        mode: HostMode,
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

`HostMode` (`open` or `sandboxed`) is the mode type of the core target of
FoundationModelsExtras (EXTRAS-12 of the TestudoApp design). It does not exist
yet. MacControl takes the same type. `mode` has no default value, so each host
states its mode. `make` throws when the mode is `open` and the process is
sandboxed (the `com.apple.security.app-sandbox` entitlement of the process is
`true`), the same as the check of MacControl.

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
      ClearList.swift
      AppleScriptRunner.swift
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
    Mail/        ...             # plus MailScripts.swift, MailboxPath.swift, MailMessageCache.swift
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
    The second product gives `ProcessRunner` and `ProcessRegistry`. Thus the
    package does not need `swift-subprocess`.
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
    if let tool = try AppleAppsTools.make(app: app, mode: hostMode) {
        builder.addGroup(named: app.rawValue, [tool])
    }
}
```

`RegistrySource` expands each `OperationTool` into verbs:
`tools.calendar.listEvents`, `tools.mail.composeMessage`. A test in the
integration package proves that no group name conflicts with a noun of
Multitool (`files`, `web`, `git`, `environment`, `shell`) or with a group that
the agent mounts (`code_context`, `skills`, `agents`).

### 10.2 FoundationModelsACPAgent

1. **Package.swift.** Add the URL dependency on FoundationModelsAppleApps.
2. **`Configuration/ToolSectionCodec.swift`.** Add the property
   `appleApps = ToolSection<AppleAppsToolOptions>.enabled(AppleAppsToolOptions())`
   to `ToolsConfiguration`. Add `appleApps` to its `CodingKeys`, to
   `init(from:)` and to `optionKeys`. `AppleAppsToolOptions` conforms to
   `ToolSectionOptions` and `KeyCheckedSection`, and has one `Bool` for each
   app. All are `false` by default. Thus an absent key, `appleApps: {}`,
   `appleApps: true` and `appleApps: false` all mount no app. The user turns
   on each app by name, in the `tools:` section:

   ```yaml
   tools:
     appleApps:
       calendar: true
       reminders: true
       mail: true
   ```

   An unknown key in the body is an error (`ConfigurationError.unknownKey`),
   because the loader checks the body against `optionKeys`.
3. **`Resources/builtin.config.yaml`.** Add the `appleApps` entry with each
   app key set to `false`. The tests of `BuiltinConfigurationFile` require
   that this file names each key of `optionKeys`.
4. **`Tools/ToolCatalog.swift`.** Add `composeAppleApps`. It calls
   `builder.addGroup(named:_:)` once for each enabled app.
5. **Info.plist.** The agent is a CLI. EventKit, Contacts and CoreLocation
   need usage strings in the `Info.plist` of the process. The agent embeds an
   `Info.plist` in the binary with the linker flag
   `-sectcreate __TEXT __info_plist`. The keys are
   `NSCalendarsFullAccessUsageDescription`,
   `NSRemindersFullAccessUsageDescription`, `NSContactsUsageDescription`,
   `NSLocationWhenInUseUsageDescription` and `NSAppleEventsUsageDescription`.
   The example of this package does the same.
6. **README.** Add one row for each app to the table in § Tools.

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
| `shortcuts` | `open` mode: none. `sandboxed` mode: Automation → Shortcuts Events (4.9). In both modes, a shortcut can ask for its own permissions. | `sandboxed` mode: same as `notes`. The correction tells the user to allow it in System Settings > Privacy & Security > Automation. |

macOS gives the permission to the responsible process. For a CLI agent that
is the terminal app or the app that started the agent. `docs/SECURITY.md` and
the README tell this.

### 11.1 Entitlements for `sandboxed` mode

A sandboxed host (Testudo) needs these entitlements and `Info.plist` keys for
each tool that it mounts. The package cannot add them. The README lists them.

| Tool | Entitlements | `Info.plist` keys |
|---|---|---|
| `calendar`, `reminders` | `com.apple.security.personal-information.calendars` | `NSCalendarsFullAccessUsageDescription`, `NSRemindersFullAccessUsageDescription` |
| `contacts` | `com.apple.security.personal-information.addressbook` | `NSContactsUsageDescription` |
| `maps` | `com.apple.security.personal-information.location`, `com.apple.security.network.client` | `NSLocationWhenInUseUsageDescription` |
| `music` | `com.apple.security.automation.apple-events`, and `com.apple.security.scripting-targets` with `com.apple.Music` → `com.apple.Music.library.read`, `com.apple.Music.playback` | `NSAppleEventsUsageDescription` |
| `shortcuts` | The same two entitlements, with `com.apple.shortcuts.events` → `com.apple.shortcuts.run` (4.9) | `NSAppleEventsUsageDescription` |
| `mail` | The same two entitlements, with `com.apple.mail` → `com.apple.mail.compose` | `NSAppleEventsUsageDescription` |

`AppleAppsTools.requiredEntitlements(for:)` returns this table for a set of
apps, so a host test can compare it with the entitlements file of the app.

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
| `ClearListTests` | Allowed names, an unknown name, and a field that is both set and cleared. |
| `AppleScriptRunnerTests` | Arguments go as `argv`, never in the script text. `-1743` gives `accessDenied`. The timeout stops the child. Uses a fake `osascript` script in a temporary directory. |
| `MailboxPathTests` | `Account/Folder/Subfolder` and `\/`. |
| `MailMessageCacheTests` | The 512-entry bound and LRU order, a stale hit falls back to a scan, `-1728` removes the entry, `move message` keeps the `id`. Uses a fake script runner. |
| `NotesIdTests` | `create note` returns the permanent id after the poll. An id that finds no note gives the `notFound` correction. A `folder` value is an id or a name. Uses a fake script runner. |
| `ChatDatabaseTests` | Reads of the synthetic `Fixtures/chat.db`, including `attributedBody`. |
| `SendMessageTests` | Delivered, failed then SMS, and not known (no second send), with a fake database and a fake sender. |
| `ShortcutsCLIBackendTests` | The output cap gives `truncated: true`. A timeout gives the `timeout` correction. Uses a fake CLI script. |
| `PackageManifestTests` | No Multitool and no `swift-otel` in the library. |
| `HostModeTests` | In `sandboxed` mode: each tool has exactly the operations of its "Sandbox" column, the op names agree with a golden for each mode, `notes` and `messages` give no tool, and no sandboxed backend uses `ProcessRunner`. `open` mode in a sandboxed process throws. |
| `InProcessAppleEventRunnerTests` | Arguments go as descriptors, never in the script text. `-1743` gives `accessDenied`. `-10004` gives the sandbox correction. Scripts run one at a time on the runner thread. Uses a fake script executor. |
| `RequiredEntitlementsTests` | `requiredEntitlements(for:)` agrees with the table of 11.1. |

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
   `DateArgument`, `ClearList`, `AppleAppsDefaults`,
   `AppleAppsConfiguration`, `URLOpener`.
3. **Script runner.** `AppleScriptRunner` over `ProcessRunner` of Extras, with
   its tests.
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
19. **Agent wiring.** The changes of 10.2 in FoundationModelsACPAgent. The
    agent is a CLI and runs in `open` mode.
20. **Host modes.** `mode` in `make` and `operationNames(in:)`, the static list
    of sandboxed operations of each tool, the check that refuses `open` mode
    in a sandboxed process, `requiredEntitlements(for:)`, and
    `HostModeTests`. Depends on EXTRAS-12 (`HostMode` in Extras; Step 1 of
    the TestudoApp roadmap, not built yet) and on items 5 to 14.
21. **In-process Apple event runner.** `AppleEventRunner`,
    `InProcessAppleEventRunner`, `ShortcutsAppleEventBackend`, and the change
    of the Music, Mail and Shortcuts backends to take the runner. A probe
    target `SandboxProbe`: a small sandboxed app with the entitlements of 11.1
    that runs each sandboxed operation. It verifies the **unverified** items
    of 4.5 and 4.8: `send` in `com.apple.mail.compose` and `sound volume`
    in `com.apple.Music.playback`. It also runs `list shortcuts` and
    `run shortcut` as a regression check of 4.9. The result updates the
    "Sandbox" columns. Depends on items 3, 9, 13 and 14.

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
7. **AppleScript through `osascript` in `open` mode. — DECIDED (2026-10-14),
   changed (2026-10-23).** Not `NSAppleScript` in `open` mode.
   `NSAppleScript` must run on one thread and cannot be cancelled. A child
   process can have a timeout and can be stopped, and `ProcessRunner` of
   Extras already does this. In `sandboxed` mode a child process cannot send
   Apple events, so that mode uses `NSAppleScript` in the process, on one
   thread, with an Apple event timeout (decision 13, 5.9).
8. **GRDB for `chat.db`. — DECIDED (2026-10-14).** The same library as
   CodeContext. The database opens read-only.
9. **MapKit for Maps. — DECIDED (2026-10-14).** Osaurus does not name its
   maps backend. MapKit needs no API key.
10. **A `clear` list clears fields. — DECIDED (2026-10-14).** Each update
    operation has `clear: [String]?`. JSON `null` cannot clear a field,
    because the `@Generable` decode changes `null` into `nil` (see 5.4). No
    change to Extras is necessary. This was open question 1.
11. **AppleScript IDs follow Osaurus. — DECIDED (2026-10-14).** See 6.2.
    This was open question 2.
12. **Two host modes; `sandboxed` registers only what works. — DECIDED
    (2026-10-23).** The same as MacControl (D128 of the TestudoApp design).
    The model never sees an operation that cannot work in the sandbox
    (principle 9, 4.11).
13. **Scripting access groups only, no temporary exceptions. — DECIDED
    (2026-10-23).** In `sandboxed` mode, Apple events go only to the access
    groups that each app publishes. Thus Notes and Messages are not
    available, and Mail has only `compose message`. Reason: App Review
    usually rejects `temporary-exception.apple-events`.

## 15. Open questions

1. **The Testudo product scope. — CLOSED (D135 of the TestudoApp design,
   2026-10-23).** The sandboxed set is enough. Notes and Messages are
   OpenTestudo-only tools. When a user asks Testudo for them, the agent says
   that the work needs OpenTestudo and gives the link (D128 (i)). The
   sandboxed set is APPLE-3 in
   `TestudoApp/features/packages/foundationmodels-appleapps.md`.
2. **Messages reads in the sandbox.** The user could give the app access to
   `~/Library/Messages` with an open panel (a security-scoped bookmark), and
   also Full Disk Access. Do both together let a sandboxed app read
   `chat.db`, `chat.db-wal` and `chat.db-shm`? This is **unverified**, and the
   user flow is poor. Until a test proves it, `messages` is not in
   `sandboxed` mode.
3. **Shortcuts as the path for the other apps. — CLOSED: no bridge (D135 of
   the TestudoApp design, 2026-10-23).** Testudo installs no user shortcuts
   to reach Notes or Messages. `run shortcut` runs only the shortcuts that
   the user made.

## 16. Out of scope

- Apps that are not in the list: Photos, Safari, Finder, Home, Files.
- GUI control of any app through the Accessibility tree (a Markdown document
  of a window, and one act operation). This is the `gui` tool of
  FoundationModelsMacControl (`FoundationModelsMacControl/plan.md`).
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
