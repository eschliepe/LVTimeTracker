# Daily roll-up: notes for wiring it into Main.vi (2026-11-08)

The roll-up is built and tested outside `Main.vi`. Claude does not edit `Main.vi`; these are the hand changes to hook it up.

## What exists

| VI | What it does |
|---|---|
| `UI Support\Day Rollup.vi` | Logic only. One row per task for a list of entries: Project, Task, Hours (decimal, 2 places, each task's total rounded to the nearest 0.25 h), Descriptions (joined with `"; "`, empty ones and exact repeats skipped, in start-time order). A running entry counts up to Now. Tested in `Tests\Run UI Support Tests.vi` ("== Day Rollup"). |
| `UI Support\Daily Rollup Dialog.vi` | Modal dialog for one day. Clicking a row copies **that task's descriptions** to the clipboard; **Copy All** copies the whole table (Project, Task, Hours, Descriptions, tab-separated, one line per task). Shows a Total. Close or the window X ends it. |

`Daily Rollup Dialog.vi` connector pane: **Project Store in** (top-left, 11) / **Project Store out** (top-right, 3), **Day** (left, 10), **Day Entries** (left, 9, `Time Entry List.ctl`), error in (8) / error out (0).

## 1. A message for it

The dialog needs the Project Store and the day's entries, which live in the main loop's `Main Loop Data`, so the button should send a message to the main loop rather than open the dialog from the event loop.

1. `Internal Message Library\Controls\Message.ctl` (the Message enum typedef): add an item, e.g. **Show Daily Rollup**. Main.vi's constants are linked to the typedef, so they pick it up.
2. Make **`Internal Message Library\Queue-Show Daily Rollup.vi`** from `Queue_Template.vit` (like the other `Queue-*.vi`), with its Message constant set to **Show Daily Rollup**, and add it to `Internal Messaging Library.lvlib`.

## 2. The button (event loop)

1. On the front panel, add a **Daily Roll-up** button (Fuse Push Button), e.g. next to the day controls. Or add a menu item to `MainMenu.rtm` if you'd rather have it in the menu.
2. New event frame **"Daily Roll-up": Value Change** (or a case in Menu Selection (User)): call `Queue-Show Daily Rollup.vi` with the event loop's queue, and wire its error out to the event structure's error output tunnel (the one that feeds `Route Error to Queue.vi`).

## 3. The frame (main loop)

New case **"Show Daily Rollup"** in the main case structure:

1. **Unbundle By Name** on Main Loop Data: `Project Store.lvclass`, `UI State.Day Entries`.
2. **Daily Rollup Dialog.vi**: Project Store in = the unbundled store, **Day** = a **Displayed Day** local variable (read, as Add/End/Modify Time Entry use), Day Entries = `UI State.Day Entries`, error in = the frame's incoming error.
3. **Bundle By Name** `Project Store out` back into Main Loop Data, and wire the dialog's error out to the frame's error output. Pass the queue, timeout and user event straight through; stop = FALSE.

`UI State.Day Entries` is always the displayed day's entries (Load Data for Month refreshes it), so no extra read is needed.

## Things to know

- **The dialog is modal and runs in the main loop**, so while it is open the main loop waits: the running timer's elapsed time on the TimeCard does not tick until the dialog closes. Queued messages are handled when it closes.
- **Clicking a row** uses the listbox's Value Change, so clicking the row that is already selected does not copy again; click another row first.
- **Panel layout is rough:** the listbox could not be resized by scripting, so the Descriptions column may be narrow. Resize the listbox and columns by hand in the panel.
- **Diagrams over one screen:** `Day Rollup.vi` (about 2640 x 525 px) and `Test Day Rollup.vi` (about 2060 x 1420 px). The per-entry grouping loop could move into a subVI if that bothers you.
- **Not driven in a test:** closing the dialog with the window X (it has a Panel Close? frame).
