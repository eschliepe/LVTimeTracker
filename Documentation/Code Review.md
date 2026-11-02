# LVTimeTracker code review

Review by Claude, October 2026, of the code on `main` at commit `a0ec770` ("Proposed Data Structures"). Every VI was checked in LabVIEW 2026 for broken code, and every working VI's block diagram was read. The bugs below were traced in the code, not reproduced by running the app.

Status column: **Open**, **Partly done** or **Done** (with the branch or commit that did it). Statuses re-checked against the code on 2026-10-31 (branch `feature/data-structures` at `05b4413`).

## Broken VIs

| VI | Status | Notes |
|---|---|---|
| `Projects/INI_class/CleanUp.vi`, `Projects/INI_class/INI_Init.vi` | Done | Broken. They belonged to an `INI.lvclass` that no longer exists and nothing called them. Deleted with the old `Projects/` folder (2026-10-25). |

Everything `Main.vi` uses is executable.

## Bugs, most important first

### 1. Today's list goes stale after starting, stopping or editing an entry — Done
**Done** on branch `feature/data-structures`: the day cache is gone. Load Data for Month re-reads the day through `UI Support\Load Day.vi` every time, and Update Time Entry replaces only the active entry's row, found by its ID (Erich's `81f91d4`, `835d1e7`, `edb3a86`). Not yet confirmed by the repro in the running app.

`Main.vi` caches each day's display rows in `Time Entries by Day` (Main Loop Data) the first time a day is loaded. "Add Time Entry", "End Time Entry", "Modify Time Entry" and "Update Time Entry" update the listbox and the INI file, but never that cache.

- **Repro:** on a day that already had entries when the app started, start a timer, step back to another day that has entries, step forward to today.
- **Result:** today is redrawn from the stale cache (the new entry is missing), then "Update Time Entry" overwrites the last row with the running entry, hiding an older one. A restart shows the right data.
- **Related:** "End Time Entry" and "Update Time Entry" always replace the *last row* of whatever day is on screen, so stopping a timer while viewing another day overwrites that day's last row on screen.
- **Related:** the "Load Data for Month" comment says the first of the month is used as a proxy for the whole month, but the code looks up the *requested day*. Empty days reload the whole month every time; days with entries never reload.
- **Fix direction:** update the day cache whenever an entry is added, stopped or edited, and only write the listbox row when the entry belongs to the day on screen.

### 2. Editing an entry from an earlier month uses the wrong file — Done
**Done** on branch `feature/data-structures`: the schema-v2 store files each entry by its own UTC start time, and `UI Support\Save Edited Entry.vi` moves an entry when its month changes (tested in `Tests\Run UI Support Tests.vi`).

"Modify Time Entry" creates a fresh `Time Entry-INI` object, whose `Initialize.vi` points it at the *current* month's file (`YYYY-MM.ini` from today's date), then reads the entry by GUID from there.

- Double-clicking a September entry in October reads from `2025-10.ini`, finds nothing, and opens Edit Time Entry with blank values (empty project/task, 1904 timestamps).
- Saving writes a duplicate section with that GUID into October's file.
- **Fix direction:** choose the month file from the entry's start time (see the proposed storage format in `Documentation/Sample Data/README.md`).

### 3. "Set to Current Day" doesn't reload the list — Done
**Done** on branch `feature/data-structures`: the Mouse Up frame now queues Load Data for Month.

The event frame writes the `Displayed Day` local variable but sends no "Load Data for Month" (and no Update/Pause), so the time card keeps the previous day's entries under today's date. Writing a local variable fires no Value Change event.

### 4. Day stepping skips or repeats a day at DST changes — Done
**Done** on branch `feature/data-structures` (2026-10-31): `Increment_Decrement by one Day.vi` steps the calendar date and builds the result like `Get Current Day as Timestamp.vi`, so they compare Equal across DST changes (checked across 2026-03-08 and 2026-11-01, and month/year rollover). Follow-up 2026-11-01: both now set the record's DST field to -1, so they return true local midnight on DST dates too (they used to return 01:00).

`Increment_Decrement by one Day.vi` adds/subtracts exactly 86,400 s. On a 23- or 25-hour day that lands an hour off midnight, so a step can skip a day or show one twice. `Main.vi` then compares the result to `Get Current Day as Timestamp.vi` with `Equal?`, which fails, so the running timer stops updating on screen.
- **Fix direction:** step the date/time record's day of month ±1 and convert back (`Date/Time To Seconds`), or re-snap to local midnight after stepping.

### 5. Unhandled messages re-run initialization — Partly done
The main loop's `"Initialize", Default` frame catches every message without its own frame.
- **Done** on branch `feature/error-logging-cases` (commit `59844e0`): `Error`, `Inactivate Project` and `Delete Time Entry` now have their own frames that pass data/error/timeout through without stopping the loop. The Error frame calls `Log Queued Error.vi`.
- **Open:** `Initialize` is still the Default frame, so any message added to the enum later without its own frame will still re-run initialization. Consider a separate Default frame that logs "unhandled message" and does nothing else. Fix notes for `Main.vi`: `Documentation/Bug Pass Notes.md`.

### 6. Errors are never reported — Partly done
- The main loop's `DeQueue State.vi` takes its `error in` from the startup error on every pass, and each frame's `error out` leaves the loop through a plain tunnel, so every pass's error is dropped. The event loop drops its errors too. Nothing shows a dialog or writes a log.
- **Worst case:** if `Create Queue.vi` / `Queue-Initialize.vi` fail at startup, `Dequeue Element` doesn't run, returns the default element (`Initialize`), and the main loop re-initializes in a tight loop forever.
- **Done:** an error logger exists (`Utilities/Error Logging/Log Error.vi`, `Log Queued Error.vi`, writing `%LOCALAPPDATA%\LVTimeTracker\Error Log.txt`) and the Error frame calls it.
- **Open:** nothing sends Error messages yet. Route errors into `Queue-Error.vi` (or call `Log Error.vi` directly) from both loops, and stop the main loop on a fatal queue error. Still true at `05b4413`. Fix notes for `Main.vi`: `Documentation/Bug Pass Notes.md`.

### 7. Making a task inactive isn't saved — Done
**Done** on branch `feature/data-structures`: the frame calls `UI Support\Set Task Active.vi`, which writes to disk (tested).

In the "Inactivate Task" frame the call to `Projects.lvclass:Write to Data Source.vi` is inside a **disabled** Diagram Disable Structure. The change is lost on exit unless the user picks Save Projects from the menu.

### 8. The projects file writer never removes anything — Done
**Done** on branch `feature/data-structures`: the schema-v2 Project Store rewrites `Projects.ini` and `Tasks.ini` in full (`Remove Project` tested in `Tests\Run Storage Tests.vi`).

`Projects-INI.lvclass:Write to Data Source.vi` overwrites `Task_N` keys but never deletes old keys or project sections. Dormant today because nothing calls `Remove Project.vi` or `Delete Project Tasks.vi`; deleted items would reappear once those are wired up. The proposed storage format (one row per task, full rewrite) removes this.

## Smaller issues

| Issue | Status | Where |
|---|---|---|
| Elapsed time can show `:60` — seconds are rounded after the minute split, so 59.6 s shows `00:00:60` | Done (`UI Support\Format Entry Row.vi` rounds first; tested) | `Calculate Time Entry.vi`, `Update Time Entry.vi` |
| No selection check: Start / double-click with no project or task selected creates an entry with an empty project or task | Done (`Start Entry.vi` refuses with error 5101; tested). Left over: the frame still stores the empty entry as active on that error, see `Documentation/Bug Pass Notes.md` | `Main.vi` "Add Time Entry" frame |
| "Default" (the template for default tasks) is listed as a real project in the project list and the Edit dialog | Done (migrated to an inactive project; default tasks now in `Settings.ini`) | `Projects.ini`, Update Project List, `Edit Time Entry.vi` |
| New projects may start inactive: the New Project dialog uses `Add Project Tasks.vi` (not `Add Project.vi`), so "Project Active?" gets the typedef default — probably False, not confirmed | Done (`New Project.vi` saves through `Save Project.vi` with Active TRUE; `865e33b`) | `New Project.vi` |
| Close dialog "Exit" leaves the running entry open in the file (End Time 0); it resumes on next start if it's in the current month. Intended? | Done: Panel Close? and the Exit menu now queue End Time Entry before Exit when an entry is active | `Main.vi` Panel Close? |
| Timestamps are written with the Windows decimal separator (`Write Key (Double)`, system decimal point) — a comma locale writes `3844268746,702599` | Done (ISO 8601 UTC strings in schema v2) | `Time Entry-INI.lvclass:Save Time Entry.vi` |
| Queue never released, user event never destroyed/unregistered | Open (fix notes: `Documentation/Bug Pass Notes.md`) | `Main.vi` |
| "Time Entry Active?" kept in two places (event-loop shift register and Main Loop Data), synced by the "Mark Active" user event | Partly done: Main Loop Data now holds `UI State.Active Entry` instead; the event loop still keeps its own flag | `Main.vi` |
| Copy-pasted, wrong VI descriptions | Done for `Increment_Decrement by one Day.vi` (2026-10-31); `Projects-INI.lvclass:Initialize.vi` was deleted with the old classes | `Projects-INI.lvclass:Initialize.vi`, `Increment_Decrement by one Day.vi` |
| Task row from right-click is computed from mouse coordinates and row height; the listbox's `Point to Row Column` method would be more robust | Open (fix notes: `Documentation/Bug Pass Notes.md`) | `Main.vi` "Make Inactive" |

## Design notes for the planned storage change

- `Time Entries` is both "one entry" and "the file it lives in": each object carries the month file it reads and writes, which is the root of bug 2. Separate the entry data (a row) from a storage object, and choose the month file from each entry's start time.
- Entries link to projects and tasks by **name**; the proposed format links by GUID with human-readable name labels (`Documentation/Sample Data/README.md`).
- The proposed format also removes bug 8 and the decimal-separator issue (ISO 8601 UTC timestamps, full row rewrites).
- Bug 1 needs its own fix regardless of storage.

## Limits of this review

The text export used for reading diagrams does not fully decode In Place Element structures or map operations, so a few internals are unverified: the "Success?" outputs of `Add Project Tasks.vi` and `Delete Project Tasks.vi`, and the inside of `Set Task Inactive.vi`. Nothing above depends on them.

## Work done since the review

On branch `feature/error-logging-cases`, all by Claude:

| Commit | Change |
|---|---|
| `59844e0` | `Main.vi`: frames for Error, Inactivate Project, Delete Time Entry; new `Log Error.vi` and `Log Queued Error.vi`; both added to the project. `Main.vi` saved as LabVIEW 2026. |
| `0607e21` | Logger VIs moved to the 4-2-2-4 connector pane; `Main.vi`'s call relinked. |
| `6ab2642` | Icons for both logger VIs. |
| `a237ad3` | Logger VI front panels converted to Fuse controls (except `Message Data`, a variant — no Fuse or System equivalent). |
